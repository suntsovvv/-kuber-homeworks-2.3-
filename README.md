# Домашнее задание к занятию «Конфигурация приложений»

### Цель задания

В тестовой среде Kubernetes необходимо создать конфигурацию и продемонстрировать работу приложения.

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
    </body>
    </html>
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
![image](https://github.com/user-attachments/assets/e79e50b9-ea06-46b1-80f1-0b1b8bb3be9d)





------

### Задание 2. Создать приложение с вашей веб-страницей, доступной по HTTPS 

Создать Deployment приложения, состоящего из Nginx, добавил configmap,ingress,service.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: homework
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
         - containerPort: 80
        volumeMounts:
        - name: api2-html
          mountPath: /usr/share/nginx/html
      
      volumes:
      - name: api2-html
        configMap:
          name: api2-html

---          
apiVersion: v1
kind: ConfigMap
metadata:
  name: api2-html
  namespace: homework
data:
  index.html: |
    <html>
    <body>
    
    <pre>
      hello tls
      --------
         \   ^__^
          \  (oo)\_______
             (__)\       )\/\
                 ||----w |
                 ||     ||
    </pre>
    </html>
    </body>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
     nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: suntsovvv.tplinkdns.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
  tls:
    - hosts:
      - suntsovvv.tplinkdns.com
      secretName: secret-tls

    
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: homework
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      name: nginx
      port: 80
      targetPort: 80
``` 
Создал tls сертификат :
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=suntsovvv.tplinkdns.com/O=suntsovvv.tplinkdns.com"
```
Создал secret в интерактивном режиме:
```bash
user@microk8s:~/kuber-homeworks-2.3$ kubectl create secret tls secret-tls --cert=tls.crt --key=tls.key
secret/secret-tls created
```
Запустил deployment:
```bash
user@microk8s:~/kuber-homeworks-2.3$ kubectl apply -f nginx-deployment.yaml 
deployment.apps/nginx-deployment created
configmap/api2-html created
ingress.networking.k8s.io/my-ingress created
service/nginx-svc created
user@microk8s:~/kuber-homeworks-2.3$ kubectl get po
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-84f484c787-f557p   1/1     Running   0          78s
user@microk8s:~/kuber-homeworks-2.3$ kubectl get svc
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-svc   ClusterIP   10.152.183.80   <none>        80/TCP    84s
user@microk8s:~/kuber-homeworks-2.3$ kubectl get secrets secret-tls 
NAME         TYPE                DATA   AGE
secret-tls   kubernetes.io/tls   2      2m31s
user@microk8s:~/kuber-homeworks-2.3$ kubectl get ingress my-ingress 
NAME         CLASS   HOSTS                     ADDRESS     PORTS     AGE
my-ingress   nginx   suntsovvv.tplinkdns.com   127.0.0.1   80, 443   105s
user@microk8s:~/kuber-homeworks-2.3$ kubectl get con
configmaps                controllerrevisions.apps  
user@microk8s:~/kuber-homeworks-2.3$ kubectl get configmaps 
NAME               DATA   AGE
api2-html          1      116s
kube-root-ca.crt   1      2d21h
user@microk8s:~/kuber-homeworks-2.3$ 
```
Проверил:
```bash
user@microk8s:~/kuber-homeworks-2.3$ curl https://suntsovvv.tplinkdns.com 
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
user@microk8s:~/kuber-homeworks-2.3$ curl https://suntsovvv.tplinkdns.com -k
<html>
<body>

<pre>
  hello tls
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
Проверил из вне :
![image](https://github.com/user-attachments/assets/5a278729-d526-4df8-9f26-fc6c136f5d45)

------
