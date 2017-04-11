Drie onderdelen:
- [NGINX](#NGINX) 
- [Let's Encrypt](#letsencrypt)
- [Fail2ban](#Fail2ban)

<a name="NGINX"/>

## NGINX:
### Locaties van mappen:
/etc/nginx                  # root nginx map  
/etc/nginx/sites-available  # locatie van sites en configs  
/etc/nginx/sites-enabled   # locatie met symlinks naar actieve configs  
## Handige commando's
`sudo service nginx restart (stop / start)`
`sudo service nginx configtest`
`sudo service nginx status`
`sudo ln -s /etc/nginx/sites-available/xxx /etc/nginx/sites-enabled/`
## install
`sudo apt-get install nginx`
`cd /etc/nginx`
`sudo openssl dhparam -out dh2048.pem 2048`
`sudo nano perfect-forward-secrecy.conf`
```
ssl ciphers "ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS";
ssl dhparam /etc/nginx/dh2048.pem;
add_header Strict-Transport-Security "max-age=31536000; includeSubdomains;";
```
Main: /etc/nginx/nginx.conf
```
user www-data;
worker_processes 4;
pid /run/nginx.pid;

events {
	worker_connections 768;
	# multi_accept on;
}
http {
	##
	# Basic Settings
	##
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	#keepalive_timeout 65;
	types_hash_max_size 2048;
	# server_tokens off;
	server_names_hash_bucket_size 64;
	# server_name_in_redirect off;
	include /etc/nginx/mime.types;
	default_type application/octet-stream;
	##
	# SSL Settings
	##
	ssl_protocols TLSv1.1 TLSv1.2;
	ssl_prefer_server_ciphers on;
	##
	# Logging Settings
	##
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;
	##
	# Gzip Settings
	##
	gzip on;
	gzip_disable "msie6";
	gzip_min_length   1100;
	gzip_vary         on;
	gzip_proxied      any;
	gzip_buffers      16 8k;
	gzip_comp_level   6;
	gzip_http_version 1.1;
	gzip_types        text/plain text/css applciation/json application/x-javascript text/xml application/xml
        	          application/rss+xml text/javascript images/svg+xml application/x-font-ttf font/opentype
                	  application/vnd.ms-fontobject;
	##
	# Virtual Host Configs
	##
	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
	include /etc/nginx/perfect-forward-secrecy.conf;
	##
	# Harden nginx against DDOS
	##
	client_header_timeout 10;
	client_body_timeout   10;
	keepalive_timeout     10 10;
	send_timeout          10;
}
```
## Empty site maken tbv testen en certificaten maken
`sudo mkdir -p /data/tempsite/www`
`sudo mkdir -p /data/tempsite/logs`
`sudo chown -R www-data:www-data /data/tempsite`
`sudo nano /etc/nginx/sites-available/tempsite`
```
server {
    listen 80 default\_server;
    root /data/tempsite/www;
    index index.php index.html index.htm;
    server\_name site.domain.com site2.domain.com site.domain2.com;
    location / {
        try\_files $uri $uri/ /index.php;
    }
    location ~ \.php$ {
        fastcgi\_pass unix:/run/php5-fpm.sock;
        include snippets/fastcgi-php.conf;
    }
}
```
Enable deze config door:
`sudo ln -s /etc/nginx/sites-available/tempsite /etc/nginx/sites-enabled/tempsite`

Nu kun je certificaten aanmaken met Let's encrypt. LE zal dan check files plaatsen op deze website, om te controleren of je de site echt beheerd.  
Als de certificaten klaar staan, kan deze tempsite weer offline gezet worden door:  
`sudo rm /etc/nginx/sites-enabled/tempsite`

## Reverse proxy config template HTTP
Maak een file aan in /etc/nginx/sites-available/  
Activeer deze door een symlink naar de enable map te maken.  
```
server {
    listen 80;
    server\_name host.domain.com host.domein2.net;
    location / {
               proxy\_pass http://&lt;internal ip van je server&gt;:80;
proxy\_redirect off;
               proxy\_set\_header Host $host;
               proxy\_set\_header X-Real-IP $remote\_addr;
               proxy\_set\_header X-Forwarded-For $proxy\_add\_x\_forwarded\_for;
               proxy\_set\_header X-Forwarded-Proto $scheme;
    }
}
```
## Reverse proxy config template HTTPS
Luister op zowel poort 80 als 443. Zorg voor een redirect op poort 80 naar poort 443.  
Maak een file aan in /etc/nginx/sites-available/  
Activeer deze door een symlink naar de site-enable map te maken.  
Eerste deel van de config staat extra auth aan in bold (kan dus ook weg gelaten worden):  
Hier wordt een user/wachtwoord bestand voor gebruikt genaamd passwords in /etv/nginx/.  
In dit bestaand staat op elke regel een user:pwd. Waarbij pwd versleuteld is door:  
`openssl passwd`
Voorbeeld regel in passwords bestand: user:3dO0JeXPQ8sGI (werkelijk: user:test)
```
server {
        listen 80;
        server_name host.domein.net www.example.com;
        return 301 https://$server_name$request_uri;
}
server {
listen 443 ssl;
    server_name host.domein.net;
    ssl on;
    ssl_certificate          /etc/letsencrypt/live/xxxxxxxxx/fullchain.pem;
    ssl_certificate_key      /etc/letsencrypt/live/xxxxxxxxx/privkey.pem;
    auth_basic "Protected";
    auth_basic_user_file passwords;
    location / {
        proxy_pass http://internal server:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
server {
    listen 443 ssl;
    server_name www.example.com;
    ssl on;
    ssl_certificate          /etc/letsencrypt/live/xxxxxxxxx/fullchain.pem;
    ssl_certificate_key      /etc/letsencrypt/live/xxxxxxxxx/privkey.pem;
    location / {
        proxy_pass http://x.x.x.x:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
<a name="letsencrypt"/>

## Lets Encrypt
### Instaleren:
`sudo apt-get install git`
`sudo git clone https://github.com/certbot/certbot /etc/letsencrypt`

### Genereren certificaten:
Je kunt meerdere host namen in 1 file genereren.  
De eerste keer wordt om een email adres gevrraagd voor recovery.  
`sudo /etc/letsencrypt/certbot-auto certonly --agree-tos --webroot -w /data/tempsite/www --cert-name ##FILENAME## -d domein1 -d domein2`

### Lijst met certificaten:
`sudo /etc/letsencrypt/certbot-auto certificates`
### Revoken van certificaten:
`sudo /etc/letsencrypt/certbot-auto revoke --cert-path /etc/letsencrypt/live/xxxxx/cert.pem`
`sudo rm -rf /etc/letsencrypt/renewal/xxxxx.conf`
`sudo rm -rf /etc/letsencrypt/live/xxxxx`
### Crontab instellen voor auto-renew elke dag om 6 uur smorgens.
`sudo crontab -e`
```
0 6 \* \* \* /etc/letsencrypt/certbot-auto renew --text &gt;&gt; /etc/letsencrypt/certbot-cron.log &amp;&amp; sudo service nginx reload
```
<a name="Fail2ban"/>

## fail2ban
Fail2ban kijkt in error logs van nginx of foutief wordt ingelogd op nginx. Als dit wel het geval is, wordt na 3 pogingen het ip adres geblokkeerd in de FW (IPTABLES).
### Install
`sudo apt-get install fail2ban -y`
### Config files
`nano /etc/fail2ban/filter.d/nginx-auth.conf`
```
[Definition]
failregex = no user/password was provided for basic authentication.*client: <HOST>
 user .* was not found in.*client: <HOST>
 user .* password mismatch.*client: <HOST>

ignoreregex = </host></host></host>
```
`mkdir -p /etc/fail2ban/jail.d`
`nano /etc/fail2ban/jail.d/nginx-auth.conf`
```
[nginx-auth]
enabled = true
filter = nginx-auth
port = http,https
logpath = /var/log/nginx*/*error*.log
bantime = 600
maxretry = 3
```
`sudo service fail2ban restart`

## Handy commands
`sudo fail2ban-client status`
`sudo fail2ban-client status nginx-auth`
`sudo fail2ban-client set nginx-auth unbanip x.x.x.x`
`sudo iptables -L -n --line-numbers`
`sudo fail2ban-client set nginx-auth addignoreip x.x.x.x of x.x.x.x/x`
`sudo fail2ban-client get nginx-auth ignoreip`

`sudo su:`
`fail2ban-client status | grep "Jail list:" | sed "s/ //g" | awk '{split($2,a,",");for(i in a) system("fail2ban-client status " a[i])}' | grep "Status\|IP list"
`

## Client side Certificate
ga naar je cert map  
Create a Certificate Authority root (which represents this server)  
Organization & Common Name: Some human identifier for this server CA.  
`openssl genrsa -des3 -out ca.key 4096`
`openssl req -new -x509 -days 365 -key ca.key -out ca.crt`

Create the Client Key and CSR  
Organization & Common Name = Person name  
`openssl genrsa -des3 -out client.key 4096`
`openssl req -new -key client.key -out client.csr`
### self-signed
`openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt`

Convert Client Key to PKCS  
So that it may be installed in most browsers.  
`openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12`

Convert Client Key to (combined) PEM  
Combines client.crt and client.key into a single PEM file for programs using openssl.  
`openssl pkcs12 -in client.p12 -out client.pem -clcerts`

Install Client Key on client device (OS or browser)  
Use client.p12. Actual instructions vary.  

Install CA cert on nginx  
So that the Web server knows to ask for (and validate) a user's Client Key against the internal CA certificate. Add the following in your website config in the sites-available folder.  
`ssl_client_certificate /path/to/ca.crt;`
`ssl_verify_client optional; # or 'on' if you require client key`