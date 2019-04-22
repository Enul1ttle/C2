### 生成ssl证书
```
#生成ssl.store文件
keytool -keystore ssl.store -storepass password -keypass password -genkey -keyalg RSA -alias cs -dname "CN=it, OU=it, O=it, L=it, S=it, C=it"
#迁移到行业标准格式 PKCS12
keytool -importkeystore -srckeystore ssl.store -destkeystore ssl.store -deststoretype pkcs12
#转换ssl.store为文件为P12
keytool -importkeystore -srckeystore server.jks -destkeystore server.p12 -deststoretype PKCS12
#使用OpenSSL的提取证书并合并
openssl pkcs12 -in server.p12 -nokeys -clcerts -out server-ssl.crt
openssl pkcs12 -in server.p12 -nokeys -cacerts -out gs_intermediate_ca.crt
#服务器ssl.crt是SSL证书，gs_intermediate_ca.crt是中级证书，合并到一起才是nginx的所需要的证书
cat server-ssl.crt gs_intermediate_ca.crt > server.crt
#提取私钥及免密码
openssl pkcs12 -nocerts -nodes -in server.p12 -out server.key
#避免重启是总是要输入私有密钥的密码
openssl rsa -in server.key -out server.key.unsecure
```
### Nginx.conf
```
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
	
upstream tomcats {
     server 被代理的服务器IP:8080;
     
	
}

server {
    listen 443 ssl;
    server_name  公网IP;
        
        ssl_certificate   /etc/pki/cn/server.crt; 
        ssl_certificate_key /etc/pki/cn/server.key;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers "EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5";
        ssl_prefer_server_ciphers on;
        ssl_session_timeout 1d;
        ssl_stapling on;
        ssl_stapling_verify on;
        add_header Strict-Transport-Security max-age=15768000;

    

    location / {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_pass https://tomcats;
    } 
  }
}
```
