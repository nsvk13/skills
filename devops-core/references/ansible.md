# Ansible Reference

## Структура проекта

```
ansible/
├── inventory/
│   ├── prod/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       ├── webservers.yml
│   │       └── databases.yml
│   └── stage/
│       └── hosts.yml
├── roles/
│   ├── common/
│   ├── docker/
│   ├── k8s-node/
│   └── monitoring/
├── playbooks/
│   ├── site.yml
│   ├── deploy.yml
│   └── upgrade.yml
└── ansible.cfg
```

### ansible.cfg — базовый
```ini
[defaults]
inventory          = inventory/prod
remote_user        = ubuntu
private_key_file   = ~/.ssh/id_ed25519
host_key_checking  = False
stdout_callback    = yaml
gathering          = smart
fact_caching       = jsonfile
fact_caching_connection = /tmp/ansible-facts
fact_caching_timeout = 3600

[ssh_connection]
pipelining  = True   # быстрее, меньше SSH handshakes
control_path = /tmp/ansible-ssh-%%h-%%p-%%r
```

---

## Inventory

### hosts.yml
```yaml
all:
  vars:
    ansible_python_interpreter: /usr/bin/python3

  children:
    k8s_masters:
      hosts:
        master-01:
          ansible_host: 10.0.0.10
        master-02:
          ansible_host: 10.0.0.11

    k8s_workers:
      hosts:
        worker-01:
          ansible_host: 10.0.0.20
        worker-02:
          ansible_host: 10.0.0.21

    monitoring:
      hosts:
        grafana-01:
          ansible_host: 10.0.0.30
```

### group_vars/all.yml
```yaml
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org

docker_version: "24.0"
```

---

## Roles — структура

```
roles/docker/
├── tasks/
│   ├── main.yml       # точка входа
│   ├── install.yml
│   └── configure.yml
├── handlers/
│   └── main.yml
├── templates/
│   └── daemon.json.j2
├── files/
│   └── some-static-file
├── vars/
│   └── main.yml       # внутренние переменные (не переопределять снаружи)
└── defaults/
    └── main.yml       # дефолты (можно переопределять)
```

### tasks/main.yml
```yaml
---
- name: Install Docker
  ansible.builtin.import_tasks: install.yml
  tags: [docker, install]

- name: Configure Docker
  ansible.builtin.import_tasks: configure.yml
  tags: [docker, config]
```

### tasks/install.yml
```yaml
---
- name: Install required packages
  ansible.builtin.apt:
    name: [ca-certificates, curl, gnupg]
    state: present
    update_cache: true
  become: true

- name: Add Docker GPG key
  ansible.builtin.apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present
  become: true

- name: Add Docker repository
  ansible.builtin.apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present
  become: true

- name: Install Docker CE
  ansible.builtin.apt:
    name: "docker-ce={{ docker_version }}*"
    state: present
  become: true
  notify: restart docker
```

### handlers/main.yml
```yaml
---
- name: restart docker
  ansible.builtin.systemd:
    name: docker
    state: restarted
    enabled: true
  become: true
```

### templates/daemon.json.j2
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "data-root": "{{ docker_data_root | default('/var/lib/docker') }}"
}
```

---

## Playbooks

### site.yml
```yaml
---
- name: Apply common configuration
  hosts: all
  roles:
    - common

- name: Setup K8s nodes
  hosts: k8s_masters:k8s_workers
  roles:
    - docker
    - k8s-node

- name: Setup monitoring
  hosts: monitoring
  roles:
    - docker
    - monitoring
```

---

## Полезные модули

```yaml
# Копировать файл
- ansible.builtin.copy:
    src: files/config.conf
    dest: /etc/app/config.conf
    owner: root
    mode: '0644'
  become: true

# Шаблон
- ansible.builtin.template:
    src: templates/app.conf.j2
    dest: /etc/app/app.conf
  notify: restart app

# Systemd
- ansible.builtin.systemd:
    name: app
    state: started
    enabled: true
    daemon_reload: true
  become: true

# Команда с идемпотентностью
- ansible.builtin.command:
    cmd: kubeadm init --config /tmp/kubeadm.yml
    creates: /etc/kubernetes/admin.conf  # не запускать если файл есть

# Shell (когда нужны пайпы)
- ansible.builtin.shell: |
    set -o pipefail
    kubectl get nodes | grep -q NotReady && exit 1 || exit 0
  args:
    executable: /bin/bash

# Block — try/rescue/always
- block:
    - ansible.builtin.command: risky-command
  rescue:
    - ansible.builtin.debug:
        msg: "Failed, running cleanup"
  always:
    - ansible.builtin.debug:
        msg: "This always runs"
```

---

## Jinja2 паттерны

```yaml
# Условие
- name: Setup swap
  when: ansible_memtotal_mb < 4096

# Цикл
- name: Create users
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: deploy,   groups: docker }
    - { name: monitor,  groups: prometheus }

# Переменная с дефолтом
dest: "{{ config_dir | default('/etc/app') }}/config.yml"

# Фильтры
"{{ some_list | join(',') }}"
"{{ hostname | upper }}"
"{{ memory_mb | int * 1024 * 1024 }}"
```

---

## Запуск

```bash
# Dry run
ansible-playbook site.yml --check --diff

# Конкретная группа / хост
ansible-playbook site.yml --limit k8s_masters
ansible-playbook site.yml --limit worker-01

# По тегу
ansible-playbook site.yml --tags docker
ansible-playbook site.yml --skip-tags monitoring

# Ad-hoc
ansible all -m ping
ansible k8s_workers -m command -a "systemctl status kubelet"
ansible all -m systemd -a "name=docker state=restarted" --become

# Проверить переменные хоста
ansible -m debug -a "var=hostvars[inventory_hostname]" master-01
```

---

## Vault — секреты

```bash
ansible-vault create inventory/prod/group_vars/all/vault.yml
ansible-vault edit vault.yml
ansible-vault encrypt_string 'mysecretpassword' --name 'db_password'

# Запуск с vault
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

```yaml
# group_vars/all/vars.yml — обычные переменные
db_host: postgres.internal

# group_vars/all/vault.yml — зашифровано
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
```

---

## Footguns

| Проблема | Решение |
|----------|---------|
| `command` не идемпотентен | Используй `creates:` или `when: not stat.stat.exists` |
| Handlers не запускаются при ошибке | `--force-handlers` или `meta: flush_handlers` |
| Facts устарели | Удали кэш `/tmp/ansible-facts` |
| `become` не работает | Проверь sudoers, или `--ask-become-pass` |
| Непонятный порядок переменных | [Документация по приоритету](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable) |