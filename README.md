# Ghost CMS on Kubernetes with FluxCD GitOps

## 📋 Опис проєкту

Цей проект демонструє розгортання **Ghost CMS** (популярної платформи для блогів) у Kubernetes кластері з використанням GitOps підходу через **FluxCD**.

### Технічний стек:
- **Застосунок**: Ghost CMS v6.10.3
- **База даних**: MySQL 8.0
- **Оператор БД**: MySQL Operator for Kubernetes (Oracle)
- **GitOps**: FluxCD
- **Оркестрація**: Kubernetes (k3d)
- **Helm**: Власний чарт для Ghost

---

## 🏗️ Структура проекту

```
Ghost/
├── apps/                      # Конфігурації застосунків
│   ├── base/                  # Базові HelmRelease для Ghost
│   └── overlays/
│       ├── production/        # Production overlay
│       └── staging/           # Staging overlay
├── charts/
│   └── ghost/                 # Власний Helm чарт для Ghost
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/         # Kubernetes маніфести
├── clusters/
│   └── cluster/               # Flux bootstrap конфігурація
├── infrastructure/            # Інфраструктурні компоненти
│   ├── base/                  # Базові налаштування
│   ├── operators/             # MySQL Operator
│   └── overlays/
│       ├── production/        # MySQL кластер для production
│       └── staging/           # MySQL instance для staging
└── README.md
```

---

## 🎯 Етапи виконання

### Етап 1: Підготовка кластера
Використовуємо локальний кластер rancher:

### Етап 2: Helm чарт (Templating)
Створено власний Helm чарт для Ghost CMS з шаблонізацією основних параметрів:
- **Image**: `ghost:6.10.3`
- **Replicas**: Конфігуруються через values.yaml
- **Resources**: CPU/Memory requests та limits
- **Autoscaling**: HPA для production
- **Ingress**: З підтримкою TLS

Основні параметри винесені в `values.yaml`:
```yaml
image:
  repository: ghost
  tag: "6.10.3"
replicaCount: 1
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

### Етап 3: База даних та Оператори
Встановлено **MySQL Operator for Kubernetes** через Flux HelmRelease.

**Створені Custom Resources для двох середовищ:**

**Staging:**
- 1 MySQL instance
- Мінімальні ресурси
- Простий режим роботи

**Production:**
- MySQL InnoDB Cluster (3 instances)
- High Availability з 2 Router instances
- Production-ready конфігурація

### Етап 4: GitOps з FluxCD
Ініціалізовано Flux у кластері:
```bash
flux bootstrap github \
  --owner=Nick-Dikiy \
  --repository=Ghost \
  --branch=master \
  --path=./clusters/cluster \
  --personal
```

**Створено два середовища:**

#### Staging Environment (`staging` namespace)
- **Репліки**: 1 (фіксовано)
- **Ресурси**: Мінімальні
- **БД**: 1 MySQL instance
- **Ingress**: `ghost.staging.local` (HTTP)

#### Production Environment (`production` namespace)
- **Репліки**: 2-5 (через HPA)
- **Ресурси**: Requests і limits встановлені
- **БД**: MySQL Cluster (3 instances + 2 routers)
- **Ingress**: `ghost.production.local` (HTTPS з TLS)
- **HPA**: Автомасштабування від 2 до 5 подів за CPU (80%)

### Етап 5: CI/CD та TLS
- **TLS**: Налаштовано self-signed сертифікати для production Ingress
- **GitOps**: Всі зміни застосовуються автоматично через Flux

---

## ✅ Підтвердження роботи

### 1. Flux HelmReleases
Всі Helm релізи у статусі Ready:

```bash
$ kubectl get helmreleases -A
NAMESPACE        NAME             AGE     READY   STATUS
mysql-operator   mysql-operator   17m     True    Helm install succeeded for release mysql-operator/mysql-operator.v1 with chart mysql-operator@2.2.6
production       ghost            3m17s   True    Helm install succeeded for release production/ghost.v1 with chart ghost@0.0.1
staging          ghost            3m17s   True    Helm install succeeded for release staging/ghost.v1 with chart ghost@0.0.1
```

✅ Всі 3 релізи (оператор + 2 застосунки) успішно встановлені

---

### 2. Flux Kustomizations
Всі оверлеї синхронізовані:

```bash
$ flux get kustomizations -A
NAMESPACE  	NAME                     	REVISION            	SUSPENDED	READY	MESSAGE
flux-system	apps-production          	master@sha1:4bde5c80	False    	True 	Applied revision: master@sha1:4bde5c80
flux-system	apps-staging             	master@sha1:4bde5c80	False    	True 	Applied revision: master@sha1:4bde5c80
flux-system	flux-system              	master@sha1:4bde5c80	False    	True 	Applied revision: master@sha1:4bde5c80
flux-system	infrastructure-operators 	master@sha1:4bde5c80	False    	True 	Applied revision: master@sha1:4bde5c80
flux-system	infrastructure-production	master@sha1:4bde5c80	False    	True 	Applied revision: master@sha1:4bde5c80
flux-system	infrastructure-staging   	master@sha1:4bde5c80	False    	True 	Applied revision: master@sha1:4bde5c80
```

✅ 6 Kustomizations синхронізовані з Git репозиторієм

---

### 3. Pods у всіх Namespace
```bash
$ kubectl get pods -A
NAMESPACE        NAME                                      READY   STATUS      RESTARTS        AGE
flux-system      helm-controller-79db8b8d69-hnsmh          1/1     Running     0               17m
flux-system      kustomize-controller-56f9d9559f-s9rmg     1/1     Running     0               17m
flux-system      notification-controller-cb59445f7-2r87q   1/1     Running     0               17m
flux-system      source-controller-5fd58f7f88-xg8pq        1/1     Running     0               17m
kube-system      coredns-6d668d687-hljhr                   1/1     Running     0               18m
kube-system      helm-install-traefik-85hzw                0/1     Completed   1               18m
kube-system      helm-install-traefik-crd-s78kn            0/1     Completed   0               18m
kube-system      local-path-provisioner-869c44bfbd-ssxqh   1/1     Running     0               18m
kube-system      metrics-server-7bfffcd44-k6mps            1/1     Running     0               18m
kube-system      svclb-traefik-d0c0d583-b9cr7              2/2     Running     0               18m
kube-system      traefik-865bd56545-j75f8                  1/1     Running     0               18m
mysql-operator   mysql-operator-596fdd5fb6-phd6p           1/1     Running     0               17m
production       ghost-56d84847f4-9jff6                    1/1     Running     0               3m14s
production       ghost-56d84847f4-b6xrk                    1/1     Running     2 (2m53s ago)   2m59s
production       mysql-0                                   2/2     Running     0               16m
production       mysql-1                                   2/2     Running     0               16m
production       mysql-2                                   2/2     Running     0               16m
production       mysql-router-78f9cd8cb6-mh8g5             1/1     Running     0               13m
production       mysql-router-78f9cd8cb6-s5m6p             1/1     Running     0               13m
staging          ghost-7d46d74c64-tlv2b                    1/1     Running     0               3m14s
staging          mysql-0                                   2/2     Running     0               16m
staging          mysql-router-56789d7df6-dhbr2             1/1     Running     0               13m
```

**Ключові спостереження:**
- ✅ **Staging**: 1 Ghost pod, 1 MySQL instance
- ✅ **Production**: 2 Ghost pods (HPA), 3 MySQL instances (Cluster), 2 MySQL Routers
- ✅ Flux компоненти працюють
- ✅ MySQL Operator запущений

---

### 4. Ingress Resources
```bash
$ kubectl get ingress -A
NAMESPACE    NAME    CLASS     HOSTS                    ADDRESS        PORTS     AGE
production   ghost   traefik   ghost.production.local   192.168.64.2   80, 443   3m17s
staging      ghost   traefik   ghost.staging.local      192.168.64.2   80        3m17s
```

✅ Staging: HTTP тільки
✅ Production: HTTP + HTTPS (TLS)

---

### 5. HorizontalPodAutoscaler (HPA)
```bash
$ kubectl get hpa -A
NAMESPACE    NAME    REFERENCE          TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
production   ghost   Deployment/ghost   cpu: 7%/80%   2         5         2          3m26s
```

✅ Production має HPA: 2-5 реплік залежно від CPU навантаження

---

### 6. MySQL InnoDB Clusters
```bash
$ kubectl get innodbcluster -A
NAMESPACE    NAME    STATUS   ONLINE   INSTANCES   ROUTERS   TYPE      AGE
production   mysql   ONLINE   3        3           2         UNKNOWN   16m
staging      mysql   ONLINE   1        1           1         UNKNOWN   16m
```

✅ **Production**: 3-node cluster з HA
✅ **Staging**: 1 instance для розробки

---

## 🔄 Self-Healing Demo

Flux автоматично відновлює видалені ресурси:

```bash
# Видалимо deployment в production
kubectl delete deployment ghost -n production

# Через 1-2 хвилини Flux відновить його
kubectl get deployment -n production
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
ghost   2/2     2            2           30s
```

---

## 🚀 Доступ до застосунку

Додайте до `/etc/hosts`:
```
192.168.64.2  ghost.staging.local
192.168.64.2  ghost.production.local
```

**Staging**: http://ghost.staging.local
**Production**: https://ghost.production.local

---

## 📚 Використані технології

| Компонент | Технологія | Версія |
|-----------|-----------|--------|
| Application | Ghost CMS | 6.10.3 |
| Database | MySQL | 8.0 |
| DB Operator | MySQL Operator | 2.2.6 |
| GitOps | FluxCD | Latest |
| Orchestration | Kubernetes | 1.31 |
| Package Manager | Helm | 3.x |
| Ingress Controller | Traefik | Latest |

---

## 📝 Висновки

Проект успішно демонструє:
1. ✅ Створення власного Helm чарту з темплейтингом
2. ✅ Використання Kubernetes Operator для управління базою даних
3. ✅ GitOps підхід з FluxCD
4. ✅ Multi-environment setup (staging/production)
5. ✅ Автомасштабування (HPA) у production
6. ✅ High Availability для бази даних
7. ✅ TLS/HTTPS для production
8. ✅ Self-healing capabilities

---

## 🔗 Репозиторій

**GitHub**: https://github.com/Nick-Dikiy/Ghost

---
