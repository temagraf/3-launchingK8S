# Домашнее задание к занятию «Запуск приложений в K8S»

## Цель задания

В тестовой среде для работы с Kubernetes необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

## Инструменты и дополнительные материалы

1. [Описание Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) и примеры манифестов
2. [Описание Init-контейнеров](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
3. [Описание Multitool](https://github.com/wbitt/Network-MultiTool)

## Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

### 1. Создание Deployment

Создаем deployment с двумя контейнерами (nginx и multitool):

```yaml
# task1/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  labels:
    app: nginx-multitool
spec:
  replicas: 1
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
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
      - name: multitool
        image: wbitt/network-multitool:latest
        env:
        - name: HTTP_PORT
          value: "8080"
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```


![image](https://github.com/temagraf/3-launchingK8S/blob/main/1-1.png)

![image](https://github.com/temagraf/3-launchingK8S/blob/main/1-2доступность%20сервисов.png)

### 2. Масштабирование

Увеличиваем количество реплик до 2:

```bash
kubectl scale deployment nginx-multitool --replicas=2
```

Результат:
```bash
kubectl get pods -l app=nginx-multitool
NAME                              READY   STATUS    RESTARTS   AGE
nginx-multitool-d9d8957c8-5sgmb   2/2     Running   0          10h
nginx-multitool-d9d8957c8-8sclb   2/2     Running   0          31s
```

![image](https://github.com/temagraf/3-launchingK8S/blob/main/1-3%20-масштаб%20до%202.png)

### 3. Создание Service

```yaml
# task1/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-multitool-service
spec:
  selector:
    app: nginx-multitool
  ports:
    - name: nginx
      port: 80
      targetPort: 80
    - name: multitool
      port: 8080
      targetPort: 8080
```

### 4. Проверка доступности

```bash
kubectl exec test-multitool -- curl nginx-multitool-service:80
```

Результат в этих строчках:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

![image](https://github.com/temagraf/3-launchingK8S/blob/main/1-4.png)


## Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

### 1. Создание Deployment с Init-контейнером

```yaml
# task2/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-init
  labels:
    app: nginx-init
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-init
  template:
    metadata:
      labels:
        app: nginx-init
    spec:
      initContainers:
      - name: init-myservice
        image: busybox:1.28
        command: ['sh', '-c', 'until nslookup nginx-init-service.default.svc.cluster.local; do echo waiting for service; sleep 2; done;']
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Состояние пода до создания сервиса:
```bash
kubectl get pods -l app=nginx-init
NAME                          READY   STATUS     RESTARTS   AGE
nginx-init-59c497c487-49lkm   0/1     Init:0/1   0          9s
```

![image](https://github.com/temagraf/3-launchingK8S/blob/main/2-1.png)

### 2. Создание Service

```yaml
# task2/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-init-service
spec:
  selector:
    app: nginx-init
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Состояние пода после создания сервиса:
```bash
kubectl get pods -l app=nginx-init
NAME                          READY   STATUS    RESTARTS   AGE
nginx-init-59c497c487-49lkm   1/1     Running   0          2m10s
```

Логи init-контейнера:
```
nslookup: can't resolve 'nginx-init-service.default.svc.cluster.local'
waiting for service
Name:      nginx-init-service.default.svc.cluster.local
Address 1: 10.152.183.216 nginx-init-service.default.svc.cluster.local
```
![image](https://github.com/temagraf/3-launchingK8S/blob/main/2-2.png) 
