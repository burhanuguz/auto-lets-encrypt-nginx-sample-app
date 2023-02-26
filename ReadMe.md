# Let's Encrypt Auto Renewal with docker compose 

Main purpose is to reverse-proxy the sample linux tweet app with nginx, and also include let's encrypt certificate automation in it.

`docker-nginx-certbot` image is used for getting and renewing the certificate. `docker-compose.yml` file has one image to generate `sample-nginx.conf` to get desired DNS. It renews certificate automatically with the help of the image, and also reloads nginx with daily healthcheck.

```bash
## Change values as desired in nginx-certbot.env file, e.g CERTBOT_EMAIL or RENEWAL_INTERVAL for let's encrypt cert
export CERTBOT_DOMAIN="DESIRED-DNS" docker compose up -d
```

#### Reference
1 - https://github.com/JonasAlfredsson/docker-nginx-certbot