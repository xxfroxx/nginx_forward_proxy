# nginx_forward_proxy
# Test Challenge:

1) Create a proxy to redirect requests for https://domain.com to 10.10.10.10 
and redirect requests for https://domain.com/resource2 to 20.20.20.20.

2) Create a forward proxy to log HTTP requests going from the internal network to the Internet including: request protocol, remote IP and time take to serve the request.

3) (Optional) Implement a proxy health check.

# Answers:

1) Handling requests with https protocol in Nginx, requires the addition of a separate module, for this reason we should use the following module: --with-http_ssl_module, but before that we must build nginx from source as indicated below.

2) A forward proxy capability in Nginx that can handle a CONNECT request, requires the addition of a separate module, for this case the following module will do the job: ngx_http_proxy_connect_module.

# Download and unzip proxy_connect module


```
git clone https://github.com/chobits/ngx_http_proxy_connect_module.git
```

# Building Nginx from source to accomplish this task:

```
wget http://nginx.org/download/nginx-1.16.0.tar.gz 
tar -zxvf nginx-1.16.0.tar.gz 
cd nginx-1.16.0.tar.gz
sudo patch -p1 < /tmp/ngx_http_proxy_connect_module-master/patch/proxy_connect_rewrite_101504.patch
# Module downloded on the step above 
./configure --sbin-path=/usr/bin/nginx --conf-path=/etc/nginx/nginx.conf --pid-path=/usr/local/nginx/nginx.pid --error-log-path=/var/log/nginx/nginx_error.log --http-log-path=/var/log/nginx/nginx_access.log --pid-path=/var/run/nginx.pid --with-http_ssl_module --add-module=/tmp/ngx_http_proxy_connect_module-master

sudo make && sudo make install
```

Add systemd service file:

`sudo vim /lib/systemd/system/nginx.service`

```
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/var/run/nginx.pid
ExecStartPre=/usr/bin/nginx -t
ExecStart=/usr/bin/nginx
ExecReload=/usr/bin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Now that Nginx is up and running with the requires modules, we should configure it.

# Configuration example and explanation

As per question #1

```
server {                                # Virtual Host for reverse proxy

    listen 443;                         # port to handle ssl protocol
    server_name domain.com;             # domain from where we handle incoming requests

    location / {                        # Any incoming requests will be redirected
        proxy_pass http://10.10.10.10;
    }

    location = /resource2 {             # Only incoming requests that match "/resource2" will be redirected
        proxy_pass http://20.20.20.20;
    }

  }

```
As per question #2

```
server {                                # Virtual Host for forward proxy
    listen       8888;                  # Port used to accept CONNECT request
    server_name  google.com;            # White listed sites
    server_name  *.google.com;          # White listed sites
    server_name  *.wp.pl;               # White listed sites
    server_name  *.rt.com;              # White listed sites
    server_name  *.netcentric.biz;      # White listed sites
    proxy_connect;                      # Module to proxy data to and from remote host
    proxy_max_temp_file_size 0;
    resolver 8.8.8.8;                   # dns resolver used by forward proxying

    location / {
      proxy_pass http://$http_host;     # Sets the protocol and address of a proxied server and an optional URI  
                                        # to which a location should be mapped.
      proxy_set_header Host $http_host; # Allows redefining or appending fields to the request header
    }
  }

```
Log configuration:

```
http {
  log_format main '$remote_addr '       									# Directive to change the format of logged messages             
                  '"$request" $status '										
                  '"$request_time" "$server_protocol"';		

  access_log /var/log/nginx/nginx_access.log main;				# Access log location and directive "main" applied
  error_log /var/log/nginx/nginx_errors.log;              # Error log location

```

Log entry breakdown:

`127.0.0.1 "CONNECT firefox.settings.services.mozilla.com:443 HTTP/1.1" 200 "60.070" "HTTP/1.1"`


```
remote_addr=                127.0.0.1 
request=                    "CONNECT firefox.settings.services.mozilla.com:443 HTTP/1.1" 
status=                     200 
request_time (miliseconds)= "60.070" 
server_protocol=            "HTTP/1.1"
```

As per question #3

```
server {
    listen 8080 default_server; 				# Port to listen for the health check

    location /nginx-health{
        access_log off;         				# No logging needed for this action
        return 200 "healthy\n"; 				# GET method response 200 OK
    }
  }

```

# Testing the forward proxy

Configure a http proxy in your browser as indicated in the image below, bear in mind that there are just few sites that you can use, those "white listed".

Check access log as follows `tail -f /var/log/nginx/nginx_access.log`


![alt text](https://github.com/xxfroxx/nginx_forward_proxy/blob/master/Screenshot%20from%202019-07-05%2003-40-15.png)

# Source for this solution

https://www.udemy.com
https://github.com/chobits/ngx_http_proxy_connect_module
https://github.com/reiz/nginx_proxy
https://hub.docker.com/r/reiz/nginx_proxy
