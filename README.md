# Docker에 Nginx, SSL적용하여 https서비스 만들기

## Securing a Containerized Django Application with Let's Encrypt

Check out the [post](https://testdriven.io/blog/django-lets-encrypt/).

## 도메인 설정
당연히 도메인이 필요하다. 프론트엔드의 IP를 도메인에 걸어줘야 하는데, A Record나 AAAA Record 둘 중의 하나로 적용해야 인증서를 발급할 수 있다.
이제 도메인에 대한 설정은 끝났다.

## https를 적용하지 않고 SSL 인증서 발급받기

### docker-compose를 실행합니다.

1. docker-compose.yml파일을 확인하고 아래 명령어를 실행하여 docker Container를 실행한다.

    ```sh
    $ docker-compose -f docker-compose.yml up -d
    ```
1. nginx와 certbot 컨테이너가 살아있는지 확인

    ```sh
    $ docker ps
    ```

### Let's Encrypt 발급

1. init-letsencrypt.sh에서 도메인, 이메일 주소, 디렉터리를 자신에 맞게 변경합니다.

1. init-letsencrypt.sh 쉘스크립트 실행하여 인증서 발급:

    ```sh
    $ sudo ./init-letsencrypt.sh // 인증서 발급  
    ```

    Test it out.

## HTTPS 적용하기

1. 이제 인증서를 발급받았으니 HTTPS를 적용할 수 있습니다.

1. nginx.conf 수정

    ```sh
    server {
        listen 80;
        server_name example.org; # 도메인으로 변경
        server_tokens off;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name example.org; # 도메인으로 변경
        server_tokens off;

        ssl_certificate /etc/letsencrypt/live/example.org/fullchain.pem; # example.org를 도메인으로 변경
        ssl_certificate_key /etc/letsencrypt/live/example.org/privkey.pem; # example.or를 도메인으로 변경
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

        location / {
            proxy_pass  <http://example.org>;
            proxy_set_header    Host                $http_host;
            proxy_set_header    X-Real-IP           $remote_addr;
            proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
        }
    }
    ```
1. 인증서를 자동으로 갱신하도록 설정하지 않으면 만료될 때마다 다시 발급을 해주어야 합니다.

1. 자동으로 인증서가 갱신되도록 docker-compose.yml 파일을 수정합니다.

    ```sh
    version: '3'

    services:
    nginx:
        image: nginx:1.15-alpine
        restart: unless-stopped
        volumes:
        - ./data/nginx:/etc/nginx/conf.d
        - ./data/certbot/conf:/etc/letsencrypt
        - ./data/certbot/www:/var/www/certbot
        ports:
        - "80:80"
        - "443:443"
        command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \\"daemon off;\\"'"
    certbot:
        image: certbot/certbot
        restart: unless-stopped
        volumes:
        - ./data/certbot/conf:/etc/letsencrypt
        - ./data/certbot/www:/var/www/certbot
        entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
    ```