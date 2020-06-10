# OTUS ДЗ WEB (Centos 7)
-----------------------------------------------------------------------
### Домашнее задание
```
Простая защита от DDOS
Цель: Разобраться в базовых принципах конфигурирования nginx. Рассмотреть хорошие\плохие практики конфигурирования, настройки ssl.
https://gitlab.com/otus_linux/nginx-antiddos-example

Следуйте инструкциям из README
Критерии оценки: 0 - не проходят тесты (описано в самопроверке)
5 - сделана базовая часть
6 - сделана продвинутая часть или обе 
```

#### Защита от ДДОС средствами nginx

Файл конфигурации - ```/etc/nginx/conf.d/default.conf```:
```
server {
    listen	 80;
    server_name  127.0.0.1 localhost;
    index index.html index.htm;
    root /usr/share/nginx/html;
    access_log  /var/log/nginx/host.access.log  main;

  location / {
            # Происходит проверка есть ли в браузере нужный нам куки, если нет, то добавляем куки в виде того адреса который запросил пользователь, чтобы потом его вернуть на тот адрес, который он запросил
            if ($http_cookie !~* "secret=otus") {
                # Этой командой добавляем в заголовок куки
                add_header Set-Cookie "originUrl=$scheme://$http_host$request_uri";
                # Делаем перенаправление на секретный раздел (страницы может и не существовать)
                return 302 $scheme://$http_host/secret;
            }
	}

	location /secret {
            # Добавляем еще один куки, секретный
            add_header Set-Cookie "secret=otus";
            
            # Если куки оригинального пользовательского адреса был присвоен, то отправить с уже 2 куками пользователя на тот адрес, который он изначально вбил в строку браузера
            if ($cookie_originUrl) {
                return 302 $cookie_originUrl;
            }
            # Если условная конструкция отдала false, то переадресовать на / сайта (создает дополнительную нагрузку, так как при этом происходит цикл до 50 переходов, поэтому лучше отключить)
            #return 302 $scheme://$http_host;
        }

    # Еще одна страница, если не будет секретного куки то доступ будет запрещен
	location /index3.html {
          if ($http_cookie !~* "secret=otus") {
          return 403;
         }
       }

        # Это правила обработки стандартных ошибок, ведут на реальные страницы
        error_page 404 /404.html;
            location = /40x.html {
        }

	    error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }

}
```

Файл конфигурации - ```/etc/nginx/nginx.conf```:
```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;


# Settings for a TLS enabled server.
#
#    server {
#        listen       443 ssl http2 default_server;
#        listen       [::]:443 ssl http2 default_server;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        ssl_certificate "/etc/pki/nginx/server.crt";
#        ssl_certificate_key "/etc/pki/nginx/private/server.key";
#        ssl_session_cache shared:SSL:1m;
#        ssl_session_timeout  10m;
#        ssl_ciphers HIGH:!aNULL:!MD5;
#        ssl_prefer_server_ciphers on;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        location / {
#        }
#
#        error_page 404 /404.html;
#            location = /40x.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#            location = /50x.html {
#        }
#    }

}
```

Важно! Необходимо чтобы веб-сервер работал с ограниченными правами, т.е. запускался например под пользователем www-data.


#### Как запустить:

```git clone git@github.com:staybox/otus_dz23.git && cd otus_dz23 && vagrant up```


#### Как проверить:

1. Через браузер (зайти на порт http://127.0.0.1:8080)

2. Через elinks локально с машины веб-сервера

3. Через curl локально с машины веб-сервера:

```Curl не умеет без прямого указания (без ключей) использовать куки```:
```
[root@web2 nginx]# curl -I localhost:80
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.16.1
Date: Tue, 09 Jun 2020 23:47:30 GMT
Content-Type: text/html
Content-Length: 145
Connection: keep-alive
Location: http://localhost/secret
Set-Cookie: originUrl=http://localhost/
```

``` Далее мы можем назначить куки и видим другой результат в заголовках```:
```
[root@web2 nginx]# curl -I --cookie 'originUrl=http://localhost:80/' localhost:80/secret 
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.16.1
Date: Tue, 09 Jun 2020 23:48:58 GMT
Content-Type: text/html
Content-Length: 145
Connection: keep-alive
Location: http://localhost:80/
Set-Cookie: secret=otus
```

```Здесь мы видим что если мы идем с нужной кукой то нам веб-сервер успешно с кодом 200 отдает страницу```:
```
[root@web2 nginx]# curl -I --cookie "secret=otus" localhost:80
HTTP/1.1 200 OK
Server: nginx/1.16.1
Date: Tue, 09 Jun 2020 23:49:11 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes
```


2. Защита от продвинутых ботов связкой nginx + java script (задание со *):

Вообще надо упомянуть что этот метод не панацея, и он также успешно обходится.

Собственную реализацию пока что не делал, однако есть проекты на базе которых можно выстроить более-менее эффективную защиту от ДДОС.

Ссылки:

https://vddos.voduy.com/

https://github.com/duy13/vDDoS-Protection

https://github.com/kyprizel/testcookie-nginx-module

https://hamsterden.ru/nginx-ddos/

https://habr.com/ru/post/139931/