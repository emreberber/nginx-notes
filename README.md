<p align="center">
  <a href="" rel="noopener">
 <img width=200px height=200px src="nginx-logo.png" alt="nginx-logo"></a>
</p>

<h3 align="center">Nginx Notes</h3>

<div align="center">

  [![Status](https://img.shields.io/badge/status-active-success.svg)]() 
  [![License](https://img.shields.io/badge/license-MIT-blue.svg)](/LICENSE)

</div>

---

<p align="center"> my nginx notes on ubuntu server
    <br> 
</p>

### 📝 Table of Contents
- [What is Nginx ?](#what_is_nginx)
- [Installing](#installing)
- [Commands](#commands)
- [Contexts](#contexts)
- [The Main Nginx Configuration File](#main_conf_file)
- [Editing nginx.conf](#nginx-conf)
- [Author](#author)
- [Useful Links](#useful_links)

### 🧐 What is Nginx ? <a name = "what_is_nginx"></a>
NGINX is open source software for web serving, reverse proxying, caching, load balancing, media streaming, and more. It started out as a web server designed for maximum performance and stability. In addition to its HTTP server capabilities, NGINX can also function as a proxy server for email (IMAP, POP3, and SMTP) and a reverse proxy and load balancer for HTTP, TCP, and UDP servers.

### 🏁 Installing <a name = "installing"></a>

```
$ sudo apt install nginx
```

### 🚀 Commands <a name = "commands"></a>

```
# edit nginx.conf
$ sudo vi /etc/nginx/nginx.conf


# determine number of cpu cores
$ lscpu


# determine number of worker connections
$ unlimit -n


# test nginx configuration file syntax before reload
$ sudo nginx -t


# reload / restart nginx
$ sudo systemctl reload nginx OR restart nginx
```

### Context <a name = "contexts"></a>

**Main Context**
- worker_procesess
- worker_rlimit_nofile


**Events Context**
- worker_connections 1024;
- multi_accept on;
- use epoll;


**HTTP Context**
- server_tokens off;
- server_names_hash_bucket_size 64;
- access_log off;
- access_log /var/log/nginx/access.log
- gzip on;
- buffers, timeouts & filehandle caching


**Buffers**
- client_body_buffer_size 10k;
- client_header_buffer_size 1k;
- client_max_body_size 8m;
- large_client_header_buffers 2 1k;


**Timeouts**
- client_body_timeout 3m;
- client_header_timeout 3m;
- keepalive_timeout 100;
- send_timeout 3m;


**Filehandle Caching**
- open_file_cache max=1000 inactive 20s;
- open_file_cache_valid 30s;
- open_file_cache_min_uses 5;
- open_file_cache_errors off;

### The Main Nginx Configuration File <a name = "main_conf_file"></a>

For many distributions, the file will be located at **/etc/nginx/nginx.conf**. If it does not exist there, it may also be at /usr/local/nginx/conf/nginx.conf or /usr/local/etc/nginx/nginx.conf

```
$ cd /etc/nginx
$ sudo cp nginx.conf nginx.conf.$(date +%Y-%m-%d)
$ lscpu
...
CPU(s):              1
...


$ unlimit -n
1024


$ sudo vi nginx.conf
```

### Editing nginx.conf <a name = "nginx-conf"></a>

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;


to


user www-data;
worker_processes 1;
worker_rlimit_nofile 10000;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;
```

- **www-data** is the user that web servers on Ubuntu (Apache, Nginx) use by default for normal operation.
The web server process can access any file that www-data can access. (su www-data)
- **worker_processes** is amount of cores you have on your setup. Some last versions calculate it automatically, use auto;
- **worker_rlimit_nofile** is the maximum file descriptors that can be opened by each worker process.


```
events {
        worker_connections 768;
        # multi_accept on;
}


to


events {
        worker_connections 1024;
        multi_accept on;
        use epoll;
}
```

- **events** context is provides the configuration file context in which the directives that affect connection processing are specified.
- **worker_conections** command tells our worker processes how many people can simultaneously be served by Nginx.
unlimit -n command gives 1024.
- **use epoll** is optimized to serve many clients with each thread, essential for linux.
- **multi_accept** on accept as many connections as possible, may flood worker connections if set too low.
- **Why is 'multi_accept' 'off' as default in Nginx?**
Probably because with on, all the worker processes are active and try to handle all of the incoming request simultaneously. When disabled, Nginx decides which child process gets to deal with the request one by one. As Nginx is very efficient at this, this probably serves most people well. Some consider it a risk to enable it, as it may flood the worker connections with requests.

```
http {


        ##
        # Basic Settings
        ##


        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;


        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;


        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        ...


to


http {


        ##
        # Basic Settings
        ##


        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        server_tokens off;


        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;


        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        ...
```

- **http** context is when configuring Nginx as a web server or reverse proxy, the "http" context will hold the majority of the configuration. This context will contain all of the directives and other contexts necessary to define how the program will handle HTTP or HTTPS connections.
The http context is a sibling of the events context, so they should be listed side-by-side, rather than nested. They both are children of the main context.
- **sendfile** by default, NGINX handles file transmission itself and copies the file into the buffer before sending it. Enabling the sendfile directive eliminates the step of copying the data into the buffer and enables direct copying data from one file descriptor to another. Alternatively, to prevent one fast connection from entirely occupying the worker process, you can use the sendfile_max_chunk directive to limit the amount of data transferred in a single sendfile() call (in this example, to 1 MB):
sendfile on;
sendfile_mac_chunk 1m;
- **tcp_nopush** directive together with the sendfile on;directive. This enables NGINX to send HTTP response headers in one packet right after the chunk of data has been obtained by sendfile().
- **tcp_nodelay** directive allows override of Nagle’s algorithm, originally designed to solve problems with small packets in slow networks. The algorithm consolidates a number of small packets into a larger one and sends the packet with a 200 ms delay. Nowadays, when serving large static files, the data can be sent immediately regardless of the packet size. The delay also affects online applications (ssh, online games, online trading, and so on). By default, the tcp_nodelay directive is set to on which means that the Nagle’s algorithm is disabled. Use this directive only for keepalive connections
tcp_nodelay on;
keepalive_timeout 65;

### ✍️ Author <a name = "author"></a>
<div align="center">

[![Status](https://img.shields.io/badge/github-emreberber-lightgrey.svg)]() 
[![Status](https://img.shields.io/badge/twitter-emreberber07-blue.svg)]() 

</div>

## 🎉 Useful Links <a name = "useful_links"></a>
-  
-
-
