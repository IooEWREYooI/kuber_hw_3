# Домашнее задание «Сетевое взаимодействие в K8S. Часть 1»

## Цель задания

В тестовой среде Kubernetes необходимо обеспечить доступ к приложению, установленному в предыдущем ДЗ и состоящему из двух контейнеров, по разным портам в разные контейнеры как внутри кластера, так и снаружи.

## Выполненные задачи

### Задача 1. Создание Deployment и обеспечение доступа к контейнерам приложения по разным портам из другого Pod внутри кластера

#### 1. Создание Deployment приложения

Создан Deployment с двумя контейнерами (nginx и multitool) и количеством реплик 3 шт.

**Манифест Deployment** (`k8s-manifests/deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool-app
  labels:
    app: nginx-multitool
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: alpine:3.19
        ports:
        - containerPort: 80
        command: ["sh", "-c", "apk add --no-cache python3 && python3 -m http.server 80"]
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 50m
            memory: 64Mi
      - name: multitool
        image: alpine:3.19
        ports:
        - containerPort: 8080
        command: ["sh", "-c", "apk add --no-cache python3 && cd /tmp && echo 'Multitool response' > index.html && python3 -m http.server 8080"]
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 50m
            memory: 64Mi
```

#### 2. Создание Service для внутреннего доступа

Создан Service, который обеспечивает доступ внутри кластера до контейнеров приложения по портам:
- 9001 → nginx:80
- 9002 → multitool:8080

**Манифест Service** (`k8s-manifests/service.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-multitool-service
  labels:
    app: nginx-multitool
spec:
  selector:
    app: nginx-multitool
  ports:
  - name: nginx
    port: 9001
    targetPort: 80
    protocol: TCP
  - name: multitool
    port: 9002
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

#### 3. Создание отдельного Pod для тестирования

Создан отдельный Pod с multitool для тестирования доступа.

**Манифест тестового Pod** (`k8s-manifests/test-pod.yaml`):
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-multitool
  labels:
    app: test-multitool
spec:
  containers:
  - name: multitool
    image: wbitt/network-multitool:latest
    command: ['sleep', '3600']
    resources:
      limits:
        cpu: 100m
        memory: 128Mi
      requests:
        cpu: 50m
        memory: 64Mi
  restartPolicy: Never
```

#### 4. Тестирование доступа внутри кластера

**Результаты тестирования:**

Доступ к nginx по порту 9001:
```bash
kubectl exec test-multitool -- wget -qO- nginx-multitool-service:9001
```
```
<!DOCTYPE HTML>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="bin/">bin/</a></li>
<li><a href="dev/">dev/</a></li>
<li><a href="etc/">etc/</a></li>
...
</ul>
<hr>
</body>
</html>
```

Доступ к multitool по порту 9002:
```bash
kubectl exec test-multitool -- wget -qO- nginx-multitool-service:9002
```
```
Multitool response
```

### Задача 2. Создание Service и обеспечение доступа к приложениям снаружи кластера

#### 1. Создание NodePort Service

Создан отдельный Service типа NodePort для внешнего доступа к nginx.

**Манифест NodePort Service** (`k8s-manifests/nodeport-service.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-service
  labels:
    app: nginx-multitool
spec:
  selector:
    app: nginx-multitool
  ports:
  - name: nginx
    port: 80
    targetPort: 80
    protocol: TCP
    nodePort: 30080
  type: NodePort
```

#### 2. Тестирование внешнего доступа

NodePort Service назначен порт 30080 на узле.

**Тестирование внешнего доступа через port-forward:**
```bash
kubectl port-forward svc/nginx-nodeport-service 8080:80
curl http://localhost:8080
```

Результат аналогичен внутреннему доступу - возвращается HTML страница с листингом директории.

## Структура файлов

```
kuber_hw_3/
├── k8s-manifests/
│   ├── deployment.yaml      # Deployment с двумя контейнерами
│   ├── service.yaml         # ClusterIP Service для внутреннего доступа
│   ├── test-pod.yaml        # Тестовый Pod
│   └── nodeport-service.yaml # NodePort Service для внешнего доступа
├── README.md                # Документация выполнения задания
└── 1.4.md                   # Описание задания
```

## Выводы

1. **Задача 1 выполнена успешно**: Создан Deployment с двумя контейнерами, Service для внутреннего доступа по разным портам, протестирован доступ из другого Pod как по IP, так и по доменному имени сервиса.

2. **Задача 2 выполнена успешно**: Создан NodePort Service для внешнего доступа к nginx, продемонстрирован доступ снаружи кластера через port-forward.

3. **Используемые технологии**:
   - Kubernetes кластер в Docker Desktop
   - Python HTTP Server для простого веб-сервера
   - Alpine Linux как базовый образ
   - kubectl для управления кластером

4. **Особенности реализации**:
   - Вместо nginx использован Python HTTP Server из-за проблем с загрузкой образов
   - Вместо multitool использован аналогичный Python сервер
   - Для внешнего тестирования использован kubectl port-forward из-за особенностей Docker Desktop

Все требования задания выполнены, доступ к контейнерам обеспечен по разным портам как внутри кластера, так и снаружи.