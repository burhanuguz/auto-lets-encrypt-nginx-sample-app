# Let's Encrypt Auto Renewal with docker-compose 

The primary purpose is to reverse-proxy the sample linux tweet app with nginx, and also include let's encrypt certificate automation in it.

- `jonasal/nginx-certbot` image is used for generating and renewing the Let's Encrypt certificate.
- In this simple architecture you will find init, nginx, and application service containers.
  - Init service: `jonasal/nginx-certbot` container itself does not have a solution for defining desired DNS in the runtime, which is why I added the same container as the init service to generate it in the runtime. `reverse-proxy.conf` file has **`${CERTBOT_DOMAIN}`** variable in it to transform in runtime, as well as needed reverse-proxy configuration inside of it under `user_conf.d` folder. The init **`${CERTBOT_DOMAIN}`** variable is needed to run docker-compose, if it is not set, compose command will not work.
  - Nginx service: Nginx container is based on **`jonasal/nginx-certbot`**. This is a special container that has certificate generation and renewal scripts inside of it, which are the most crucial parts of this task. Init containers generated `reverse-proxy.conf` as volume. The usual conf.d folder was not used because `jonasal/nginx-certbot` has already some configuration in it. `user_conf.d`, a special volume is used by the container `jonasal/nginx-certbot` to put user configurated nginx conf files.
  - Application service: This is the sample application used. It is even built with `docker-compose.yml`

```yaml
version: '3'

services:
  ## Init service to generate reverse-proxy.conf
  init:
    image: jonasal/nginx-certbot:latest
    environment:
      - CERTBOT_DOMAIN=${CERTBOT_DOMAIN:?err}
    volumes:
      - ./user_conf.d:/user_conf.d
      - nginx_conf:/etc/nginx/user_conf.d
    entrypoint: |
      /bin/bash -c "envsubst '$${CERTBOT_DOMAIN}' < /user_conf.d/reverse-proxy.conf > /etc/nginx/user_conf.d/reverse-proxy.conf"
  ## Nginx reverse proxy service
  nginx:
    image: jonasal/nginx-certbot:latest
    restart: unless-stopped
    environment:
      - CERTBOT_EMAIL=${CERTBOT_EMAIL:?err}
    env_file:
      - ./nginx-certbot.env
    ports:
      - 80:80
      - 443:443
    volumes:
      - nginx_secrets:/etc/letsencrypt
      - nginx_conf:/etc/nginx/user_conf.d
    depends_on:
      - init
    ## Healtcheck reloads nginx every day to serve with new cert if it is renewed
    healthcheck:
      test: ["CMD", "nginx", "-s", "reload"]
      interval: 24h
      start_period: 40s # Start healthcheck command after 40 sec
  ## Linux tweet simple app container
  application:
    build: ./linux_tweet_app
    image: application:v1
    depends_on:
      - nginx
    restart: unless-stopped

volumes:
  nginx_secrets:
  nginx_conf:
```


```bash
## Change values as desired in nginx-certbot.env file, e.g CERTBOT_EMAIL or RENEWAL_INTERVAL for let's encrypt cert
export CERTBOT_DOMAIN="DESIRED-DNS"
export CERTBOT_EMAIL="EMAIL-ADDRESS"
docker compose up -d
```

#### Reference
1 - https://github.com/JonasAlfredsson/docker-nginx-certbot