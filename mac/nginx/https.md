```shell
cd /usr/local/etc/nginx

openssl genrsa -out server.key 1024

openssl req -new -key server.key -out server.csr
# 随便填一个

openssl rsa -in server.key.org -out server.key

openssl x509 -req -in server.csr -out server.crt -signkey server.key -days 3650

vim nginx.conf
server {
        listen       443 ssl;
        server_name  api.fanli.com;

        ssl_certificate     server.crt;
        ssl_certificate_key  server.key;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            #root   html;
            #index  index.html index.htm;
            proxy_pass http://localhost:8080;
        }
    }
    
    
    
    sudo nginx -s reload
```

