---
title: "Hosting Hackmd and Nginx inside Docker containers"
date: 2019-03-12T23:45:32+08:00
draft: false
lastmod: 2019-03-12T23:45:32+08:00
tags: ["docker", "hackmd", "nginx"]
categories: ["Tech"]
---

It's my side project for learning networking and docker. :) 

The goal is to run both Nginx to Hackmd in Docker containers.

<!--more-->

# Introduction

Hackmd can be easily run in the container thanks to the developers from HackMD for dockerizing the entire project already. The part for setting up Nginx, interconnecting containers, and properly setting up the port and stuff to expose the containers, is what makes this project interesting.

Let's begin our journey!

Make sure that you have docker and docker-compose installed.

# Hackmd

## Installation

Running self-hosted Hackmd is super easy

* Get the docker stuff that is already prepared for you by running `git clone https://github.com/hackmdio/codimd-container.git`
* `cd docker-hackmd`, and `run docker-compose up` (In future runs, `docker-compose up -d` to run the containers in the background)

## Update
‌
Taken from the official documentation.

```
cd docker-hackmd ## enter the directory
git pull ## pull new commits
docker-compose pull ## pull new containers
docker-compose up ## turn on
```

## Backup

### Where is the data?

* If we trace the `docker-compose` file of HackMD, we can see that for the `database`, there is a volume associated with it.
* After running `docker-compose up`, we can run `docker volume ls`to inspect what volumes are mounted now. 
* After knowing the volume's name, we can use `docker volume inspect [name]` to see more, including the location that the volume is actually storing its data!  
    * [`sudo cd` won't work](https://askubuntu.com/questions/291666/why-doesnt-sudo-cd-var-named-work). Use `sudo -i` and then `cd`

### Backup 

`docker-compose exec database pg_dump hackmd -U hackmd > backup.sql`

### Restore 
`cat backup.sql | docker exec -i $(docker-compose ps -q database) psql -U hackmd`

# Nginx

Resources [1](https://hackprogramming.com/how-to-setup-subdomain-or-host-multiple-domains-using-nginx-in-linux-server/) [2](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04)

We will build our own Nginx image for container, so we can get the bash shell, logs, and be able control lots of stuff easily. The tradeoff will be huge image size though...

The `dockerfile` for Nginx. Build it with `docker build -t nginx .`

```
FROM ubuntu:bionic

# silent debconf warnings
ARG DEBIAN_FRONTEND=noninteractive

# Update and install software packages
RUN apt-get update
RUN apt-get upgrade -y
RUN apt-get install -y apt-utils man 
RUN apt-get install -y locales sudo wget curl htop
RUN apt-get install -y vim 
RUN apt-get install -y nginx 
RUN mkdir /var/run/sshd

RUN locale-gen en_US.UTF-8
RUN locale-gen zh_TW.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

# Nginx config file for test.henrybear327.com
# Notice that the symbolic link for linking must be absolute path!
COPY test.henrybear327.com /etc/nginx/sites-available/test.henrybear327.com
RUN sudo ln -s /etc/nginx/sites-available/test.henrybear327.com /etc/nginx/sites-enabled/test.henrybear327.com

# basic auth file
COPY .htpasswd /etc/nginx/conf.d

# expose port 80 for HTTP (443 for HTTPS)
EXPOSE 80

# start nginx
CMD ["nginx", "-g", "daemon off;"]‌
```

Nginx config file and basic auth details is explained below.

## [Basic auth](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/)

The basic auth works by creating a username-password file by using `htpassed` tool. After creating the file, we just need to add it to Nginx's configuration file. 

* Install `apache2-utils`
* Create users `sudo htpasswd -c .htpasswd Henry` 
    * `-c` means create file, so only use it on the first run!)
* Watch out for the location you put the file in the container. If you put it at `/root`, Nginx won't be able to read it, since its permission is `700`. 
    * Internal server error of `500` will occur since the system can't open the file
    * Digging from the log, which is usually kept at `/var/log/nginx`, is a good way good way to debug!

## Hosting on Virtual Host

For every virtual host that you want to host your website, create a dedicated file for it! For example, if I want to have a website at `test.henrybear327.com`, we need to create a config file with name `test.henrybear327.com`

Sample config file:

```
server {
    server_name test.henrybear327.com;

    auth_basic           "Test site";
    auth_basic_user_file /etc/nginx/conf.d/.htpasswd; 

    location / {
        proxy_pass http://docker-hackmd_app_1:3000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
    }
}
```

There are several things in this file that required attention. 

The first thing is the `proxy_pass`. Because when running `docker-compose`, it will automatically create a new network! So the Nginx must be on the same network as HackMD (will be discussed about later).

Another thing is that the address of the container we will be connecting to. It's not `localhost:3000`, it's the container's name!

## Starting HackMD and Nginx containers

If you have done the previous HackMD and Nginx setup correctly, we can now start our containers! We will start the HackMD first, and then connect Nginx to HackMD's network, and exposing the 80 or 443 port to the outside world!

```
‌sudo docker-compose up -d # start the hackmd containers
sudo docker run -d --rm --name nginx --net=docker-hackmd_backend -p 80:80 nginx # start the Nginx container
```

The part for starting the Nginx container is a little bit complicated. 

As mentioned before, `docker-compose` will create its own network. So after starting it, we need to use `sudo docker network ls` to check the network name, and specify it in the `--net` argument.

`-p 80:80` or `-p 443:443` depends on your connection type. 

## Stopping HackMD and Nginx containers
‌
Simply... stop it :)

```
sudo docker-compose down
sudo docker stop nginx
```

# Sidenotes

* Use `docker container exec <container> nginx -s reload` to reload Nginx if you have changed some config file
* Whatever changes you do to the Nginx config files, run sudo nginx -t to perform a check before reloading it

## [Nginx with Cloudflare SSL](https://support.cloudflare.com/hc/en-us/articles/115000479507)

After messing with Let's encrypt for quite a while, I decided to simply use Cloudflare's origin certificate...

The following passage is taken from this post... [Great explanation about Cloudflare origin vs Let's encrypt Certificate](https://www.reddit.com/r/sysadmin/comments/awtpcr/differences_among_cloudflare_origin_certificates/)

> If you use Cloudflare to front your website, there's no real point to continually renewing Let's Encrypt certs every few months. The end user never sees your server cert and doesn't have to trust it, so the long-lived origin cert is probably less work.

### Setup

* Have your site hide behind cloudflare's. 
    ![Cloudflare Crypto page](https://i.imgur.com/1MfWJV0.png)
    ![Cloudflare CA post](https://i.imgur.com/A7p75UL.png)
* Generate the origin certificate
* Update dockerfile and nginx config file‌
* Update `dockerfile` and `nginx config file`
    * ‌New test.henrybear327.com config file
        ```
        server {
            listen   443;

            ssl    on;
            ssl_certificate    /etc/ssl/certs/henrybear327.com.pem;
            ssl_certificate_key    /etc/ssl/private/henrybear327.com.key;

            server_name test.henrybear327.com;

            auth_basic           "Test";
            auth_basic_user_file /etc/nginx/conf.d/.htpasswd; 

            location / {
                proxy_pass http://docker-hackmd_app_1:3000/;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
            }
        }
        ```
    * New `dockerfile`
        ```‌
        FROM ubuntu:bionic

        # silent debconf warnings
        ARG DEBIAN_FRONTEND=noninteractive

        # Update and install software packages
        RUN apt-get update
        RUN apt-get upgrade -y
        RUN apt-get install -y apt-utils man 
        RUN apt-get install -y locales sudo wget curl htop
        RUN apt-get install -y vim 
        RUN apt-get install -y nginx 
        RUN mkdir /var/run/sshd

        RUN locale-gen en_US.UTF-8
        RUN locale-gen zh_TW.UTF-8
        ENV LANG en_US.UTF-8
        ENV LANGUAGE en_US:en
        ENV LC_ALL en_US.UTF-8

        # Nginx config file for test.henrybear327.com
        # Notice that the symbolic link for linking must be absolute path!
        COPY test.henrybear327.com /etc/nginx/sites-available/test.henrybear327.com
        RUN sudo ln -s /etc/nginx/sites-available/test.henrybear327.com /etc/nginx/sites-enabled/test.henrybear327.com

        # basic auth file
        COPY .htpasswd /etc/nginx/conf.d

        # SSL stuff
        COPY henrybear327.com.key /etc/ssl/private
        COPY henrybear327.com.pem /etc/ssl/certs

        # expose port 80 for HTTP (443 for HTTPS)
        EXPOSE 443

        # start nginx
        CMD ["nginx", "-g", "daemon off;"]
        ```

Turn your `SSL` to `full(strict)` and set `Always Use HTTPS` to `on`!

Well done! Now the site is on HTTPS!
