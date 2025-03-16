---
title: Nginx와 Certbot을 활용한 SSL 적용
author: pjh5365
date: 2025-3-16 16:23:00 +0900
categories: [DevOps, Nginx]
tags: [devops, nginx]
image:
  path: /assets/img/nginx.png
  alt: Nginx
---

[저번 게시글](https://pjh5365.github.io/posts/nginx%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%98%EC%97%AC-%EB%8F%84%EB%A9%94%EC%9D%B8-%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0/)에서 Nginx를 활용하여 도메인을 적용하는 방법에 대해 알아보았다. 이번에는 Certbot을 사용하여 이전에 설정하였던 도메인의 프로토콜을 `HTTP` 에서 `HTTPS` 로 변경하는 방법에 대해 알아보겠다.
## Nginx 설정
우선 이전 게시글에서 설정한 Nginx 설정 파일에 `Certbot`이 SSL 인증서 발급을 위해 접근할 수 있도록 하는 설정을 추가한다.
```bash
server {
    listen 80;
    server_name www.pjh5365.com;

    # 80포트로 들어오면 8080으로 전달
    location / {
        proxy_pass http://52.79.61.157:8080/; # 백엔드 서버 주소
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # 인증서 발급을 위한 설정 추가
    location /.well-known/acme-challenge/ {
        root /usr/share/nginx/html;
        allow all;
    }
}
```
그리고 이전에 실행했던 Nginx를 종료하고 443 포트와 인증서 위치를 볼륨마운트에 추가한 도커 명령으로 Nginx를 재실행한다.
```bash
docker rm -f nginx # 이전에 실행중이던 Nginx 강제 종료

# 443 포트와 SSL 인증서 발급을 위한 볼륨마운트 추가 (certs, html)
docker run -d -p 80:80 -p 443:443 \
  --name nginx \
  -v /docker/nginx/conf:/etc/nginx/conf.d \
  -v /docker/nginx/certs:/etc/nginx/certs \
  -v /docker/nginx/html:/usr/share/nginx/html \
  nginx:alpine
```
## Certbot
Nginx 설정이 마무리되었으므로 Certbot을 도커로 실행하여 인증서를 발급받는다. (`Certbot` 은 90일마다 새로 갱신해주어야 한다.)
```bash
# nginx를 볼륨마운트한 경로를 볼륨마운트에 추가하고 원하는 도메인을 설정한다.
docker run -it --rm \
  -v /docker/nginx/certs:/etc/letsencrypt \
  -v /docker/nginx/html:/usr/share/nginx/html \
  certbot/certbot certonly --webroot \
  -w /usr/share/nginx/html \
  -d www.pjh5365.com
```
## SSL 적용
이제 Nginx 설정 파일에 다시 접근하여 모든 `HTTP` 요청을 `HTTPS` 요청으로 리다이렉션 시키는 설정을 추가하고 Nginx를 재실행하면 `HTTPS` 로 정상적으로 접근할 수 있음을 확인할 수 있다.
```bash
# HTTP -> HTTPS 리다이렉션
server {
    listen 80;
    server_name www.pjh5365.com;

    # 인증서 발급을 위한 설정 추가
    location /.well-known/acme-challenge/ {
        root /usr/share/nginx/html;
        allow all;
    }

    # 모든 HTTP 요청을 HTTPS로 리다이렉션
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name www.pjh5365.com;

    # Certbot으로 생성한 SSL 인증서 경로
    ssl_certificate /etc/nginx/certs/live/www.pjh5365.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/live/www.pjh5365.com/privkey.pem;

    # 443포트로 들어오면 8080으로 전달
    location / {
        proxy_pass http://52.79.61.157:8080/; # 백엔드 서버 주소
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
```bash
docker restart nginx
```
### 실행 화면
![](/assets/img/2025-3-16-ssl/1.png)
