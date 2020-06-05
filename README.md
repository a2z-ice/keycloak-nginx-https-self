<pre><code>
vim /etc/nginx/conf.d/multi_host.conf
server {
    listen          80;
    server_name     example1.com;

    location / {
        root    /var/www/websites/;
        index   index.html  index.htm;
    }
}
server {
    listen          80;
    server_name     example2.com;

    location / {
        root    /var/www/websites/;
        index   index2.html  index2.htm;
    }
}

##ssl for keycloak configuration
vim /etc/nginx/conf.d/ssl.conf

server {
    listen 443 http2 ssl;
    listen [::]:443 http2 ssl;

    server_name idp.oss.net.bd;

    ssl_certificate "/home/cent-node141/certificate/server.crt";
    ssl_certificate_key "/home/cent-node141/certificate/keyfile.key";
   
    
    location / {

     if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' '$http_origin' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, PATCH, DELETE';
        #
        # Custom headers and headers various browsers *should* be OK with but aren't
        #
        add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range';
        #
        # Tell client that this pre-flight info is valid for 20 days
        #
        add_header 'Access-Control-Max-Age' 1728000;
        add_header 'Content-Type' 'text/plain; charset=utf-8';
        add_header 'Content-Length' 0;
        return 204;
      }     

      proxy_pass http://127.0.0.1:8080;
      proxy_set_header	Host			$host;
      proxy_set_header	X-Real-IP		$remote_addr;
      proxy_set_header	X-Forwarded-For		$proxy_add_x_forwarded_for;
      proxy_set_header	X-Forwarded-Host	$host;
      proxy_set_header	X-Forwarded-Server	$host;
      proxy_set_header	X-Forwarded-Port	$server_port;
      proxy_set_header	X-Forwarded-Proto	$scheme;
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}

</code></pre>
# Nginx Reverse Proxy to add HTTPS for Keycloak

## Why?

tl;dr: Telling minikube to use a locally started OIDC provider is hard.

The goal was to setup an OIDC provider for a local minikube installation. However,
the kubernetes `apiserver` does not support non-https OIDC Issuers. When using a self-signed
certificate `apiserver` accepts a CA. So the goal was to set this up with a rootCA that can then be 
trusted by the API server

## Requirements

Docker, Docker-compose, openssl
Optional: Minikube if you also want to start up minikube with OIDC pointed to the proxied keycloak

## How to use?

It should be relatively easy to reproduce:

### 1. Create CA
Run 
```sh
./create_ca.sh
```

### 2. Create and sign Certificates for nginx
Run 
```sh
./issue_cert.sh
```
This also adds the virtual box magic IP "10.0.2.2" as a subjectAltName to the certificate

### 3. build nginx and start up the network
```sh
docker-compose build && docker-compose up
```

### 4. (Optional) Add client in keycloak
Open https://localhost:8443 and sign in with user `admin` and password `pass`. Then create a client called `kubernetes-cluster`.

### 5. (Optional) start up minikube pointed to keycloak
Run
```sh
./start_minikube.sh
```

You can now use OpenID connect to authenticate against your minikube cluster. Keep in mind that the token will be considered invalid if the
issuer doesn't exactly match the way it was specified when starting up the `apiserver`. This means if you get your token from `localhost` it will not have `10.0.2.2` in the issuer uri. To get around that you can simply create an alias IP. There is a script (`./add_ip.sh`) which works on macOS. On linux you can probably do the same with `iptables`.


