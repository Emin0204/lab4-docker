# Контрольные вопросы

## Блок 1. Базовые понятия

### 1. Что такое Kubernetes и зачем нужен?
Система оркестрации контейнеров. Автоматизирует развёртывание, масштабирование, самовосстановление и обновления приложений в кластере.

### 2. Docker вручную vs Kubernetes

| Docker | Kubernetes |
|---|---|
| Одна машина | Кластер |
| Нет самовосстановления | Автоперезапуск подов |
| Нет масштабирования | Реплики, HPA |
| Нет балансировки | Service |
| Ручные обновления | Rolling update |

### 3. Примеры, где оркестрация экономит время
- **Микросервисы:** один `kubectl apply` вместо ручного запуска на N серверах.
- **CronJob:** ночные задачи по расписанию без внешних скриптов.

### 4. Компоненты кластера

| Компонент | За что отвечает |
|---|---|
| kube-apiserver | REST API, точка входа |
| etcd | Хранилище состояния |
| kube-scheduler | Выбор ноды для пода |
| kube-controller-manager | Приводит реальное состояние к желаемому |
| kube-proxy | Реализует Service |
| kubelet | Агент на ноде, следит за контейнерами |
| containerd | Запуск контейнеров |
| CoreDNS | DNS кластера |
| CNI (kindnet) | Сеть между подами |

---

## Блок 2. Pod и YAML

### 5. Обязательные поля YAML для пода
`apiVersion`, `kind`, `metadata` (имя), `spec`.

### 6. Пример YAML

```yaml
apiVersion: v1          # версия API
kind: Pod               # тип ресурса
metadata:
  name: my-app-pod      # имя пода
spec:
  containers:
  - name: my-container  # имя контейнера
    image: nginx:latest # образ
    ports:
    - containerPort: 80 # порт

Если пропустить `apiVersion`/`kind` → ошибка распознавания; без `metadata.name` → ошибка пустого имени; без `spec` → под ничего не запустит.

---

## Блок 3. Сервисы и Ingress

### 7. Service. ClusterIP vs Headless
**Service** — стабильный IP/DNS для группы подов.

- **ClusterIP** — внутренний IP, балансирует трафик.
- **Headless** (`clusterIP: None`) — DNS отдаёт IP всех подов, без балансировки (для БД/StatefulSet).

### 8. NodePort vs LoadBalancer
- **NodePort:** порт 30000–32767 на каждой ноде → Service → под.
- **LoadBalancer:** внешний облачный балансировщик → NodePort → Service → под.

### 9. Порты и ограничения NodePort
Диапазон 30000–32767. Ограничения: порт занят на всех нодах, некрасивый URL, нет L7/TLS.

### 10. Когда NodePort, когда LoadBalancer
- **NodePort** — dev/test, локально.
- **LoadBalancer** — прод, публичный IP, TLS, отказоустойчивость.

### 11. Ingress
L7-маршрутизатор HTTP/HTTPS. Решает проблему множества NodePort/LoadBalancer: один вход, маршрутизация по хосту/пути, TLS.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80

## Блок 4. Контроллеры, ConfigMap/Secret, kubectl, Namespaces

### 12. Deployment vs StatefulSet vs DaemonSet

| | Deployment | StatefulSet | DaemonSet |
|---|---|---|---|
| **Цель** | Stateless (веб, API) | Stateful (БД) | Агенты на каждой ноде |
| **Поды** | Взаимозаменяемы | Упорядоченные, стабильные имена | По одному на ноду |
| **Привязка** | Нет | PV на каждый под | Жёстко к узлу |

При 3 репликах:

- **Deployment** — 3 взаимозаменяемых пода;
- **StatefulSet** — `-0`, `-1`, `-2` со своими томами;
- **DaemonSet** — реплики не задаются вручную, по поду на ноду.

### 13. Job vs CronJob

- **Job** — одноразовая задача (миграция, бэкап).
- **CronJob** — по расписанию (ночной отчёт).

Проверка: `kubectl get jobs/cronjobs`, логи — `kubectl logs job/<name>`.

### 14. ConfigMap и Secret. Два способа передачи

- **ConfigMap** — несекретные данные, **Secret** — секреты (base64).
- **Переменные окружения** (`envFrom`/`env`).
- **Файлы через том** (`volumes` + `volumeMounts`).

`env` — для простых параметров; том — для конфиг-файлов.

### 15. YAML ConfigMap и Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  APP_MODE: "production"
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=
### 15. YAML ConfigMap и Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  APP_MODE: "production"
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=
Пароли не в ConfigMap, потому что ConfigMap не шифруется, доступен всем в namespace, попадает в логи и дампы.

### 16. Команды kubectl

| Задача | Команды |
|---|---|
| Посмотреть | `get pods/nodes/all`, `describe`, `logs` |
| Создать/обновить | `apply -f`, `create -f`, `scale`, `rollout restart` |
| Удалить | `delete pod/<name>`, `delete -f` |
| Отладить | `exec -it`, `port-forward`, `events`, `describe` |

### 17. Namespaces

Логическое разделение кластера: dev/stage/prod, команды, приложения. Даёт изоляцию имён, квоты, RBAC.

### 18. Создать namespace и работать в нём

`kubectl create namespace dev`

`kubectl apply -f pod.yaml -n dev`

`kubectl config set-context --current --namespace=dev`

---

## Блок 5. Probes, хранилища, самовосстановление, обновления

### 19. Probes

- **livenessProbe** — жив ли контейнер, иначе перезапуск.
- **readinessProbe** — готов ли принимать трафик, иначе убрать из Service.
- **startupProbe** — для медленного старта, блокирует liveness/readiness до готовности.

### 20. PV и PVC

**PV** — ресурс хранилища (NFS, диск, облако). **PVC** — запрос пода на хранилище. PVC биндится к PV (или через StorageClass), под монтирует PVC как том.

### 21. Самовосстановление

- Упал под → ReplicaSet создаёт новый.
- Упал контейнер → kubelet перезапускает по `restartPolicy`.
- Упал узел → поды пересоздаются на других нодах после таймаута.

### 22. Rolling update

Постепенная замена подов без простоя. Параметры: `maxSurge`, `maxUnavailable`, `strategy.type: RollingUpdate`.

### 23. Запуск обновления и наблюдение

`kubectl set image deployment/my-app my-container=nginx:1.25`

`kubectl rollout status deployment/my-app`

`kubectl rollout history deployment/my-app`

`kubectl rollout undo deployment/my-app`

`kubectl get pods -w`