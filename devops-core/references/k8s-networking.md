# K8s Networking Reference

## DNS Resolution Order
```
Pod → kube-dns (CoreDNS) → <svc>.<ns>.svc.cluster.local
```

## NetworkPolicy — deny all + allow specific
```yaml
# Сначала deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# Потом allow конкретно
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 3000
```

## Ingress с cert-manager
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
```

## Service Types
- ClusterIP — внутри кластера
- NodePort — 30000-32767, для dev/debug
- LoadBalancer — внешний IP (MetalLB в bare metal)
- ExternalName — CNAME к внешнему сервису

## Debugging сети
```bash
# DNS внутри пода
kubectl run debug --image=busybox --rm -it -- nslookup kubernetes.default

# Connectivity check
kubectl run debug --image=nicolaka/netshoot --rm -it -- bash

# Посмотреть endpoints
kubectl get endpoints <service>

# Calico диагностика
calicoctl get networkpolicy -A
calicoctl node status
```