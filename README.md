# Домашнее задание к занятию «Конфигурация приложений»

### Цель задания

В тестовой среде Kubernetes необходимо создать конфигурацию и продемонстрировать работу приложения.

------

### Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (например, MicroK8s).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым GitHub-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/configuration/secret/) Secret.
2. [Описание](https://kubernetes.io/docs/concepts/configuration/configmap/) ConfigMap.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment приложения и решить возникшую проблему с помощью ConfigMap. Добавить веб-страницу

Создал Deployment приложения, состоящего из контейнеров nginx и multitool, так же создал ConfigMap для добавления переменной окружения контейнера multitool:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cm-deployment
  namespace: homework
  labels:
    app: cm-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cm-deployment
  template:
    metadata:
      labels:
        app: cm-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
         - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
         - containerPort: 8080
        env:
        - name: HTTP_PORT
          valueFrom:
            configMapKeyRef:
              name: myconfigmap
              key: key1

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigmap
  namespace: homework
data:
  key1: "8080"
```


Проверил, что pod стартовал и оба конейнера работают:
```bash
user@microk8s:~/kuber-homeworks-2.3$ kubectl apply -f deployment-cm.yaml 
deployment.apps/cm-deployment created
configmap/myconfigmap created
user@microk8s:~/kuber-homeworks-2.3$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
cm-deployment-6b5c49df8b-l2t8h   2/2     Running   0          11s
user@microk8s:~/kuber-homeworks-2.3$ 
```

Добавил ConfigMap для создания web страницы:
```yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-html
  namespace: homework
data:
  index.html: |
    <html>
    <body>
    
    <pre>
      hello
      --------
         \   ^__^
          \  (oo)\_______
             (__)\       )\/\
                 ||----w |
                 ||     ||
    </pre>
    </html>
    </body>
```
Добавил service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: cm-de-svc
  namespace: homework
spec:
  selector:
    app: cm-deployment
  ports:
    - protocol: TCP
      name: nginx
      port: 80
      targetPort: 80
    - protocol: TCP
      name: multitool
      port: 8080
      targetPort: 8080
  externalIPs:
    - 192.168.0.105

```
Проверяю

```bash
user@microk8s:~/kuber-homeworks-2.3$ kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/cm-deployment-cc4c756c6-48847   2/2     Running   0          76m

NAME                TYPE        CLUSTER-IP       EXTERNAL-IP     PORT(S)           AGE
service/cm-de-svc   ClusterIP   10.152.183.103   192.168.0.105   80/TCP,8080/TCP   18h

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cm-deployment   1/1     1            1           76m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/cm-deployment-cc4c756c6   1         1         1       76m
user@microk8s:~/kuber-homeworks-2.3$ kubectl get configmaps 
NAME               DATA   AGE
api-html           1      50m
kube-root-ca.crt   1      2d16h
myconfigmap        1      18h
user@microk8s:~/kuber-homeworks-2.3$ curl 192.168.105
<html>
<body>

<pre>
  hello
  --------
     \   ^__^
      \  (oo)\_______
         (__)\       )\/\
             ||----w |
             ||     ||
</pre>
</html>
</body>
```
На моем машрутизаторе настроен ddns, настроил проброс на машину с k8s и проверил в браузере из вне:




------

### Задание 2. Создать приложение с вашей веб-страницей, доступной по HTTPS 

1. Создать Deployment приложения, состоящего из Nginx.
2. Создать собственную веб-страницу и подключить её как ConfigMap к приложению.
3. Выпустить самоподписной сертификат SSL. Создать Secret для использования сертификата.
4. Создать Ingress и необходимый Service, подключить к нему SSL в вид. Продемонстировать доступ к приложению по HTTPS. 
4. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

------

### Правила приёма работы

1. Домашняя работа оформляется в своём GitHub-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------
