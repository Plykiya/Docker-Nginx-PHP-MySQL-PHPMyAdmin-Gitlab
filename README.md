# Docker-Nginx-PHP-MySQL-PHPMyAdmin-Gitlab
Docker running PHP-FPM, MySQL, using Nginx as a reverse proxy for Gitlab and PHPMyAdmin

### Description
We'll be making use of Docker's network feature and using Nginx as a reverse proxy for Gitlab, PHPMyAdmin, and PHP-FPM. You don't have to self-host Gitlab, I'm doing it for fun. If you are, make sure your system meets [Gitlab's minimum requirements](https://docs.gitlab.com/ee/install/requirements.html) or else Gitlab installation will fail.

### Installing Docker and Docker Compose
Find your OS, download [Docker](https://docs.docker.com/install/) and [Docker Compose](https://docs.docker.com/compose/install/) for it. Make sure to add yourself to the docker user group.

### Docker Images to pull
- [nginx](https://hub.docker.com/_/nginx/)
- [mysql](https://hub.docker.com/_/mysql/)
- [phpmyadmin/phpmyadmin](https://hub.docker.com/r/phpmyadmin/phpmyadmin/)
- [gitlab/gitlab-ce](https://hub.docker.com/r/gitlab/gitlab-ce/)
- [php:7.4-fpm](https://hub.docker.com/_/php)

### My Set up
I'm doing this on a fresh install of Ubuntu 18.04, have already pre-configured my domain's DNS records.

Final Project Tree
```
.
├── gitlab
│   └── blablabla persistent gitlab storage, will retain logins and branches and stuff
├── mysql
│   └── blablabla persistent mysql storage
└── Website
  ├── docker-compose.yml
  ├── nginx
  │   ├── blablabla bunch of normal nginx stuff  
  │   ├── nginx.conf
  │   └── conf.d
  │       └── default.conf
  │       └── gitlab.trap.fashion.conf
  │       └── phpmyadmin.trap.fashion.conf
  └── www
      ├── index.php
      └── hello.html

```
### Installation Steps (Assuming you've already pulled the images and installed Docker + Docker Compose)
Setting up Gitlab
1. Run Nginx through Docker, copy the nginx folder from the container onto your host machine so that we can make changes to the Nginx config. Stop and remove the Nginx container.
```
docker run -p 80:80 -d nginx
docker ps
docker cp containername:/etc/nginx /home/plykiya/website
```
2. Set up a Docker network and launch Gitlab. We'll use persistent volume storage, retaining any logins and data through launches.
```
docker network create trap-network
docker run -d --hostname gitlab.trap.fashion --network trap-network --restart unless-stopped --volume /home/plykiya/gitlab/config:/etc/gitlab --volume /home/plykiya/gitlab/logs:/var/log/gitlab --volume /home/plykiya/gitlab/data:/var/opt/gitlab gitlab/gitlab-ce
```

3. In the Nginx conf.d folder, we add in our reverse proxy config for Gitlab and then start Nginx. The IP will be the IP of the Gitlab docker container on the local network. Use docker inspect containername if you need to find out.
```
cd /home/plykiya/website/nginx/conf.d
sudo vi gitlab.trap.fashion.conf
```
The config file. The client max body size is necessary when initially setting up your Gitlab repo.
```
server {
  listen 80;
  listen [::]:80;

  server_name gitlab.trap.fashion;

  location / {
    proxy_pass http://172.18.0.2/;
    proxy_set_header Host $host;
    proxy_buffering off;
    proxy_set_header X-Real-IP $remote_addr;
    client_max_body_size 100m;
  }
}
```

4. Run Nginx, Gitlab should be accessible at the server_name specified.
```
docker run --network trap-network -v /home/plykiya/website/www:/usr/share/nginx/html:ro -v /home/plykiya/website/nginx/conf.d:/etc/nginx/conf.d/ -d -p 80:80 nginx
```

Setting up PHP-FPM, PHPMyAdmin, Docker-Compose
1. My docker-compose.yml file looks like so:
```
version: '3.7'
networks:
    trap-network:
        driver: bridge
services:
    php:
        image: php:7.4-fpm
        container_name: trap-php
        networks:
            - trap-network
        environment:
            TIMEZONE: EST
        volumes:
            - '/home/plykiya/website/www:/usr/share/nginx/html'
    nginx:
        image: nginx
        container_name: trap-nginx
        networks:
            - trap-network
        depends_on:
            - php
            - gitlab
            - phpmyadmin
        restart: always
        environment:
            PHP_FPM_SERVER: php:9000
        volumes:
            - '/home/plykiya/website/www:/usr/share/nginx/html:ro'
            - '/home/plykiya/website/nginx/conf.d:/etc/nginx/conf.d/'
        ports:
            - '80:80'
    db:
        image: mysql
        container_name: trap-mysql
        networks:
            - trap-network
        command: --default-authentication-plugin=mysql_native_password
        environment:
            MYSQL_ROOT_PASSWORD: supersupersuperdupersecretpassword
        restart: always
        volumes:
            - '/home/plykiya/mysql:/var/lib/mysql'
    phpmyadmin:
        image: phpmyadmin/phpmyadmin
        container_name: trap-phpmyadmin
        networks:
            - trap-network
        depends_on:
            - db
        restart: always
    gitlab:
        image: gitlab/gitlab-ce
        container_name: trap-gitlab
        networks:
            - trap-network
        volumes:
            - '/home/plykiya/gitlab/config:/etc/gitlab'
            - '/home/plykiya/gitlab/logs:/var/log/gitlab'
            - '/home/plykiya/gitlab/data:/var/opt/gitlab'
        restart: always
```

2. Add Nginx reverse proxy config for PHPMyAdmin, edit Gitlab config if you set it up, edit default config for PHP-FPM
PHPMyAdmin config (Just change the proxy pass name on the Gitlab one to gitlab)
```
server {
  listen 80;
  listen [::]:80;

  server_name phpmyadmin.trap.fashion;

  location / {
    proxy_pass http://phpmyadmin/;
    proxy_buffering off;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```
Edit the index line to look for index.php, and paste the php block.
```
index index.php index.html;

location ~ ^/.+\.php(/|$) {
        fastcgi_index  index.php;
        fastcgi_pass php:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
```
3. Stop and remove Nginx/Gitlab containers running if they are. Navigate to the directory with docker-compose.yml and run Docker Compose. 
```
docker-compose up -d
```
You may need to update Gitlab permissions if you're getting 502 errors, restart both Gitlab and Nginx if you do. Congrats, it should all work now.
```
sudo docker exec trap-gitlab update-permissions
sudo docker restart trap-gitlab
sudo docker restart trap-nginx
```
