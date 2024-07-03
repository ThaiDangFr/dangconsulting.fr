---
layout: post
title: TLS simple sidecar container
date: 2024-06-21 12:00:00-0000
description: Utiliser un sidecar pour gérer le TLS avec des clefs existantes
tags: tls sidecar kubernetes
categories: education
giscus_comments: true
related_posts: false
toc:
  beginning: true
---

Container sidecar
=================
Le but d'un container sidecar est de rajouter des fonctionnalités au container principal au sein d'un Pod Kubernetes.
On utilise ici la propriété qu'au sein d'un même Pod, les container peuvent utiliser les mêmes ressources réseau et stockage.

Workshop sidecar TLS avec une paire de clef créée manuellement
==============================================================

### Pré-requis

ns.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: poc-sidecar-tls
```

création d'un namespace dédié
```bash
kubectl apply -f ns.yaml
kubectl config set-context --current --namespace=poc-sidecar-tls
```

### Container A : un serveur web sur le port 80

docker-nginx-hello-world/Dockerfile
```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
```

docker-nginx-hello-world/index.html
```html
<html>
  <head><title>Hello world</title></head>
  <body>Hello world</body>
</html>
```

création et push de l'image
```bash
cd docker-nginx-hello-world
docker build -t thaidangfr/nginx-hello-world:v1.0 .
docker push thaidangfr/nginx-hello-world:v1.0
```


### Container B : un sidecar tls sur le port 443 qui proxyfie vers le port 80 (et donc le container A)

docker-sidecar-nginx-tls/Dockerfile
```bash
FROM nginx:latest
COPY default.conf /etc/nginx/conf.d/default.conf
```

docker-sidecar-nginx-tls/default.conf
```nginx
server {
    listen 443 ssl;
    ssl_certificate       /etc/nginx/ssl/tls.crt;
    ssl_certificate_key   /etc/nginx/ssl/tls.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # allow large body sizes
    client_max_body_size 32m;

    # increase client body buffer for performance.
    client_body_buffer_size 128k;

    ssl_session_cache shared:SSL:20m;

    access_log /dev/stdout;
    error_log /dev/stdout info;

    location / {
        # proxy to upstream application
        proxy_pass http://127.0.0.1:80;

        # don't use http 1.0 so keepalive enabled by default
        proxy_http_version 1.1;

        # prevent client from closing keepalive
        proxy_set_header Connection "";

        # if the proxied server does not receive anything within this time, the connection is closed
        proxy_send_timeout 86400s;
        proxy_read_timeout 86400s;

        # don't write client body to docker file system
        proxy_request_buffering off;
    }
 }
```

création et push de l'image
```bash
cd docker-nginx-sidecar-tls
docker build -t thaidangfr/nginx-sidecar-tls:v1.0 .
docker push thaidangfr/nginx-sidecar-tls:v1.0
```

### Création de la paire de clef tls
```bash
# create a self-signed certificate
openssl req -x509 -newkey rsa:2048 -keyout tls.key -out tls.crt -nodes -subj '/CN=nginx-sidecar-tls-svc'

# create a TLS secret
kubectl create secret tls tls-cert --cert=tls.crt --key=tls.key
```

### Création du pod et du service

deploy.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poc-sidecar-tls-deploy
  namespace: poc-sidecar-tls
  labels:
    app: poc-sidecar-tls
spec:
  replicas: 1
  selector:
    matchLabels:
      app: poc-sidecar-tls
  template:
    metadata:
      labels:
        app: poc-sidecar-tls
    spec:  
      containers:
        - name: nginx
          image: thaidangfr/nginx-hello-world:v1.0
          imagePullPolicy: Always
          ports:
            - name: http-port
              containerPort: 80
        - name: nginx-tls
          image: thaidangfr/nginx-sidecar-tls:v1.0
          imagePullPolicy: Always
          ports:
            - name: https-port
              containerPort: 443
          volumeMounts:
            - name: tls-cert
              mountPath: /etc/nginx/ssl              
      volumes:
        - name: tls-cert
          secret:
            secretName: tls-cert
```

svc.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: poc-sidecar-tls-svc
  namespace: poc-sidecar-tls
spec:
  selector:
    app: poc-sidecar-tls
  ports:    
    - port: 443
      targetPort: https-port
```

installation du Deployment (qui va engendrer un ReplicaSet), puis du Service
```bash
kubectl apply -f deploy.yaml
kubectl apply -f svc.yaml
```

### Test

on vérifie que l'on accède bien au serveur en https
```bash
kubectl proxy
curl http://localhost:8001/api/v1/namespaces/poc-sidecar-tls/services/https:poc-sidecar-tls-svc:443/proxy/
```
