Сначала склоним репо certbot в home
```sh
cd ~
git clone https://github.com/certbot/certbot
cd certbot
```
Далее нам нужно подготовить наш сервер к acme-challenge, копируем это в нужно server директорию в /etc/nginx/sites-available/default

```sh
location ~ ^/(.well-known/acme-challenge/.*)$ {
    proxy_pass http://127.0.0.1:9999/$1;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Далее генерируем сам сертификат, переходим в папку с certbot и запускаем

```sh
./certbot-auto --agree-tos --renew-by-default --standalone --standalone-supported-challenges http-01 --http-01-port 9999 --server https://acme-v01.api.letsencrypt.org/directory certonly -d mydomain.com -d www.mydomain.com
```

При генерации сертификата может возникнуть ошибка:
```sh
OSError: Command /home/deploy/.local/...ncrypt/bin/python2.7 - setuptools pkg_resources pip wheel failed with error code 1
```
Чтобы ее решить достаточно просто проставить локали
```sh
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"
```
Далее включаем ssl в нужной секции server в конфиге ngnix
```sh
listen 443;
server_name domain.com www.domain.com;

ssl on;
ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;
```

И добавляем редирект с версии http -> https
```sh
server {
  listen       80;
  server_name  domain.com www.domain.com;
  return       301 https://domain.com$request_uri;
}
```

Сертификат работает, теперь нужно настроить автоапдейт сертификата

Делаем
```sh
crontab -e
```
и добавляем строчку
```sh
0 0 * * 2 /home/deploy/certbot/certbot-auto renew --post-hook "/etc/init.d/nginx restart" --noninteractive --no-self-upgrade --force-renew >> /var/log/certbot.log 2>&1
```

Последний шаг, разрешим башу выполнять некоторые судо команды без пароля
Запускаем
```sh
sudo visudo
```
и добавляем в самый конец (если юзер отличен от deploy заменить на имя юзера)
```sh
deploy ALL=(ALL) NOPASSWD:SETENV:/home/deploy/certbot/certbot-auto
deploy ALL=(ALL) NOPASSWD:SETENV:/home/deploy/.local/share/certbot/bin/certbot
deploy ALL=(ALL) NOPASSWD:SETENV:/home/deploy/.local/share/letsencrypt/bin/letsencrypt
```


