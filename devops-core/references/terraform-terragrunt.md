# Terraform & Terragrunt Reference

## Структура проекта

### Простой проект (один env)
```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
└── modules/
    ├── vm/
    ├── network/
    └── k8s/
```

### Multi-env без Terragrunt
```
terraform/
├── modules/
│   ├── vm/
│   └── network/
└── environments/
    ├── prod/
    │   ├── main.tf
    │   └── backend.tf
    └── stage/
        ├── main.tf
        └── backend.tf
```
Минус: дублирование backend и provider блока в каждом env → решается Terragrunt.

---

## Terraform — паттерны

### versions.tf — фиксируй версии
```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    # пример — адаптируй под свой провайдер
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # пример S3 backend — адаптируй под свой
  backend "s3" {
    bucket = "tf-state-prod"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### Модуль — минимальный шаблон
```hcl
# modules/vm/variables.tf
variable "name"      { type = string }
variable "cores"     { type = number; default = 2 }
variable "memory_gb" { type = number; default = 2 }
variable "disk_gb"   { type = number; default = 20 }
variable "subnet_id" { type = string }
variable "image_id"  { type = string }

# modules/vm/main.tf
# тело ресурса зависит от провайдера
resource "cloud_instance" "this" {
  name      = var.name
  subnet_id = var.subnet_id
}

# modules/vm/outputs.tf
output "internal_ip" { value = cloud_instance.this.private_ip }
output "id"          { value = cloud_instance.this.id }
```

### for_each вместо count (почти всегда)
```hcl
# count — проблема: удаление из середины сдвигает индексы
resource "cloud_instance" "bad" {
  count = 3
  name  = "node-${count.index}"
}

# for_each — стабильные ключи, безопасное удаление
locals {
  nodes = {
    "node-a" = { cores = 2, memory_gb = 4 }
    "node-b" = { cores = 4, memory_gb = 8 }
  }
}

resource "cloud_instance" "good" {
  for_each = local.nodes
  name     = each.key
  # each.value.cores, each.value.memory_gb
}
```

### lifecycle — защита критичных ресурсов
```hcl
resource "cloud_database" "db" {
  lifecycle {
    prevent_destroy       = true       # terraform destroy упадёт с ошибкой
    ignore_changes        = [password] # не трогать если изменилось снаружи
    create_before_destroy = true       # zero-downtime замена
  }
}
```

### locals для DRY
```hcl
locals {
  env    = terraform.workspace
  prefix = "myapp-${local.env}"
  common_tags = {
    environment = local.env
    managed-by  = "terraform"
  }
}
```

---

## Terraform — команды

```bash
# Базовый workflow
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# Конкретный ресурс
terraform plan  -target=module.vm.cloud_instance.this
terraform apply -target=module.vm.cloud_instance.this

# Импорт существующего ресурса
terraform import cloud_instance.node <instance-id>

# State операции (осторожно)
terraform state list
terraform state show <resource>
terraform state mv <old> <new>  # переименование без пересоздания
terraform state rm <resource>   # убрать из state без удаления ресурса

# Workspaces
terraform workspace new stage
terraform workspace select prod

# Форматирование и валидация
terraform fmt -recursive
terraform validate
```

---

## Terragrunt — зачем нужен

Решает главные проблемы Terraform при multi-env:
1. **Дублирование backend/provider** — пишешь один раз в `root.hcl`
2. **Зависимости между модулями** — `dependency` блок вместо remote state вручную
3. **DRY inputs** — общие переменные наследуются вниз по иерархии

---

## Terragrunt — структура

```
infra/
├── terragrunt.hcl        # root: backend + provider (наследуется всеми)
├── prod/
│   ├── env.hcl           # env-level переменные
│   ├── network/
│   │   └── terragrunt.hcl
│   ├── vm/
│   │   └── terragrunt.hcl
│   └── k8s/
│       └── terragrunt.hcl
└── stage/
    ├── env.hcl
    └── ...
```

### root terragrunt.hcl
```hcl
locals {
  env_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  env      = local.env_vars.locals.env
}

remote_state {
  backend = "s3"  # адаптируй под свой бэкенд
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite"
  }
  config = {
    bucket = "tf-state-${local.env}"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    region = local.env_vars.locals.region
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<EOF
# адаптируй под свой провайдер
provider "aws" {
  region = "${local.env_vars.locals.region}"
}
EOF
}
```

### env.hcl
```hcl
locals {
  env    = "prod"
  region = "us-east-1"
}
```

### Модуль terragrunt.hcl с dependency
```hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/vm"
}

dependency "network" {
  config_path = "../network"
}

inputs = {
  name      = "prod-node"
  subnet_id = dependency.network.outputs.subnet_id
  cores     = 4
  memory_gb = 8
}
```

### Terragrunt — команды

```bash
# Один модуль
terragrunt plan
terragrunt apply

# Все модули в директории (с учётом зависимостей)
terragrunt run-all plan
terragrunt run-all apply

# Без подтверждения (CI)
terragrunt run-all apply --terragrunt-non-interactive
```

---

## Когда Terraform, когда Terragrunt

| Ситуация | Выбор |
|----------|-------|
| 1-2 environment, простой проект | Terraform + папки или workspaces |
| 3+ environments, много модулей | Terragrunt |
| Зависимости между модулями | Terragrunt |
| Командная работа, нужен стандарт | Terragrunt |