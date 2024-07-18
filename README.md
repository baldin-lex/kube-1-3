# Домашнее задание к занятию «Запуск приложений в K8S» - Балдин

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.

Ошибка решается добавлением еще одного порта, так как свободного уже нет (nginx занял). 

2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.

До:

```bash
baldin@kube-test:~/kube-1-3$ kubectl get pods -n netology
NAME                                   READY   STATUS    RESTARTS   AGE
nginx-multitool-64d8c3cdff-mm7t5       1/1     Running   0          15m
```

После:

```bash
baldin@kube-test:~/kube-1-3$ kubectl get pods -n netology
NAME                                   READY   STATUS    RESTARTS   AGE
nginx-multitool-64d8c3cdff-mm7t5       2/2     Running   0          43m
nginx-multitool-64d8c3cdff-npg2c       2/2     Running   0          38s
```

4. Создать Service, который обеспечит доступ до реплик приложений из п.1.

Манифест Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-multitool-netology
  namespace: netology
spec:
  selector:
    app: web
  ports:
    - name: http-port
      port: 80
      protocol: TCP
      targetPort: 80
    - name: alter-port
      port: 22020
      protocol: TCP
      targetPort: 22020
```

5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.

```bash
baldin@kube-test:~/kube-1-3$ kubectl exec -n netology multitool-app -- curl nginx-multitool-netology:80
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   712  100   712    0     0   2726      0 --:--:-- --:--:-- --:--:--  2738
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

```bash
baldin@kube-test:~/kube-1-3$ kubectl exec -n netology multitool-app -- curl nginx-multitool-netology:22020
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0WBITT Network MultiTool (with NGINX) - nginx-multitool-64d8c3cdff-npg2c - 10.1.58.173 - HTTP: 22020 , HTTPS: 443 . (Formerly praqma/network-multitool)
```

------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.

Манифест:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-init-deploy
  namespace: netology
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
      containers:
      - name: nginx
        image: nginx:1.19.2
      initContainers:
      - name: init-busybox
        image: busybox
        command: ['sleep', '50']
```

```bash
baldin@kube-test:~/kube-1-3$ kubectl apply -f nginx_init_deploy.yml
deployment.apps/nginx-init-deploy created
baldin@kube-test:~/kube-1-3$ kubectl get pods -n netology
NAME                                   READY   STATUS     RESTARTS      AGE
nginx-multitool-64d8c3cdff-mm7t5       2/2     Running    2 (25m ago)   4h
nginx-multitool-64d8c3cdff-npg2c       2/2     Running    2 (25m ago)   4h
multitool-app                          1/1     Running    0             37m
nginx-init-deploy-53f6fcc78g-fx3v6     0/1     Init:0/1   0             53s
```

3. Создать и запустить Service. Убедиться, что Init запустился.

```bash
baldin@kube-test:~/kube-1-3$ kubectl apply nginx_init_svc.yaml
service/nginx-init-svc created
```

4. Продемонстрировать состояние пода до и после запуска сервиса.

```bash
baldin@kube-test:~/kube-1-3$ kubectl get pods -n netology
NAME                                   READY   STATUS     RESTARTS      AGE
nginx-multitool-64d8c3cdff-mm7t5       2/2     Running    2 (37m ago)   4h
nginx-multitool-64d8c3cdff-npg2c       2/2     Running    2 (37m ago)   4h
multitool-app                          1/1     Running    0             49m
nginx-init-deploy-9856ff5c27-z2p25     1/1     Running    0             71s
```

Под запущен.
