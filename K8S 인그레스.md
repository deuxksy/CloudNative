# Learning Kubernetes

[toc]

## 12 Ingress 

### 12.1 Ingress 컨트롤러 생성

- nginx Ingress 컨트롤러 생성
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/cloud/deploy.yaml
```

- 생성 확인
```
kubectl get pods -n ingress-nginx \
  -l app.kubernetes.io/name=ingress-nginx --watch
```
ingress-nginx-controller pod가 running이면 정상 설치 완료

### 12.2 Deployment 생성

- nginx / goapp deployment 생성

```{yaml}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.7.9
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goapp-deployment
  labels:
    app: goapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: goapp
  template:
    metadata:
      labels:
        app: goapp
    spec:
      containers:
      - name: goapp-container
        image: chungsju/goapp
        ports:
        - containerPort: 8080
```

### 12.3 Service 생성

```{yaml}
apiVersion: v1
kind: Service
metadata:
  name:  nginx-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx

---
apiVersion: v1
kind: Service
metadata:
  name:  goapp-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: goapp
```

### 12.4 Ingress 생성

```{yaml}
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx-goapp-ingress
spec:
  tls:
  - hosts:
    - nginx.acorn.com
    - goapp.acorn.com
    secretName: acorn-secret
  rules:
  - host: nginx.acorn.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-lb
          servicePort: 80
  - host: goapp.acorn.com
    http:
      paths:
      - path: /
        backend:
          serviceName: goapp-lb
          servicePort: 80
```

### 12.5 Ingress 조회

```{bash}
kubectl get ingress

NAME                  HOSTS                             ADDRESS   PORTS     AGE
nginx-goapp-ingress   nginx.acorn.com,goapp.acorn.com             80, 443   15s
```

> Ingress 가 완전히 생성되기 까지 시간이 걸립니다. 2~5분 소요

다시 조회 합니다

```{bash}
kubectl get ingress

NAME                  HOSTS                             ADDRESS          PORTS     AGE
nginx-goapp-ingress   nginx.acorn.com,goapp.acorn.com   35.227.227.127   80, 443   13m
```

### 12.6 /etc/hosts 파일 수정 (C:\Windows\System32\drivers\etc\hosts 파일 수정)

```{bash}
sudo vi /etc/hosts

35.227.227.127 nginx.acorn.com goapp.acorn.com
```

### 12.7 서비스 확인

```{bash}
$ curl http://goapp.acorn.com
hostname: goapp-deployment-d7564689f-6rrzw

$ curl http://nginx.acorn.com
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

```

### 12.8 HTTPS 서비스 (TLS OffLoad)

- 인증서 생성 및 인증서 secret 등록

```{bash}
openssl genrsa -out server.key 2048

openssl req -new -x509 -key server.key -out server.cert -days 360 -subj /CN=nginx.acorn.com

kubectl create  secret tls acorn-secret --cert=server.cert --key=server.key
```

- 테스트

```{bash}
$ curl -k https://nginx.acorn.com

$ curl -k https://goapp.acorn.com
```





