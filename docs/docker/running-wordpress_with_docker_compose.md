---
layout: post
title: Running WordPress with Docker compose
nav_order: 1
parent: Docker
permalink: docker/running-wordpress-with-docker-compose
---

# Running WordPress with Docker compose

Installing the LAMP stack to run WordPress can be time-consuming and difficult to replicate between multiple systems. Docker containers make running MySQL and the WordPress server simple. Docker-compose simplifies managing configuration to run multiple docker containers. Data and content persistence can be achieved by using the volume mounts.

Installing WordPress on your System will require:

- PHP: Version 7.3 or greater.
- Web Server: Apache or Nginx
- Database: MySQL version 5.0.15 or greater or any version of MariaDB.

This is also known as the LAMP stack. The stack typically consists of the Linux operating system, the Apache HTTP Server, the MySQL relational database management system, and the PHP programming language. You can use MAMP for MacOS or WAMP for Windows to get started with the LAMP stack. But before you go through all that which can take a while, the easier way would be to run Wordpress and MySQL as Docker Containers.

If you are not familiar with Docker and have not already installed Docker and Docker compose, please follow these tutorials to install them in your system and go through the quick introduction provided at these URLs:

- [Install Docker](https://docs.docker.com/v17.09/engine/installation/#supported-platforms)
- [Install Docker Compose](https://docs.docker.com/compose/install/)

Once you have Docker and Docker Compose installed, running Wordpress will be the same process in all platforms!

Create a new directory for your project and add `./docker-compose.yml` file with the following content:

```yml
version: '3.3'

services:
   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: unless-stopped
     environment:
       MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: ${MYSQL_PASSWORD}
   wordpress:
     depends_on:
       - db
     image: wordpress:latest
#     volumes:
#      - ./wp-content:/var/www/html/wp-content
     ports:
       - 8000:80
     restart: unless-stopped
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
       WORDPRESS_DB_NAME: wordpress
volumes:
    db_data: {}
```

Create one more file `.env` which will have passwords for MySQL

```bash
MYSQL_ROOT_PASSWORD=wordpress
MYSQL_PASSWORD=wordpress
```

To start Wordpress run: `$ docker-compose up`

Go to http://localhost:8000, you must have WordPress running with the installation page, follow the steps as shown in page and configure your site as well as the admin user credentials. Once done, add a new post with some random content as well.

Docker compose created a Docker volume named db_data which will store the database contents from /var/lib/mysql, then pulled docker images for mysql:5.7 and wordpress:latest and ran those containers.

### Data Persistence

If you are familiar with Docker you must know that each time docker container restarts it is identical to the source image, if files were added to a running container, those changes are lost. We are using the db_data volume mount so that when containers restart we will not lose the data. It will use the same volume, which exists in the local filesystem, each time the container starts.

If you would like to list the volume and find out some details about it run:

`$ docker volume ls`

`$ docker volume inspect <volume_name>`

Let\'s stop the containers and start them again. You can either hit Ctrl+C or run:

`$ docker-compose stop`

`$ docker-compose start`

If you go back to the webpage, you must notice that the admin user still existed and the new post that was created still there, which means the database configurations were intact.

#### Content Persistence

In the WordPress dashboard, go to Appearance and install a new theme. Now let\'s restart the containers:

`$ docker-compose restart`

You must notice that the installed theme is gone, that is because we are not storing /var/www/html/wp-content directory in the filesystem, and each time you restart the wp-content is identical the initial worpress:latest image. Let\'s fix that:

First, while the containers are still running, open a new terminal and run:

`$ docker ps`

You must see container *wp_wordpress_1 running, copy that name and go the project directory. Inside the project directory run:
`$ docker cp CONTAINER_NAME:/var/www/html/wp-content .`

This will copy the current wp-content directory into the local directory. We can stop the containers now

`$ docker-compose stop`

Next, open the `docker-compose.yml` file we created earlier and uncomment the lines for wp-content volumes mount :

```yml
     volumes:
      - ./wp-content:/var/www/html/wp-content
```

Start the containers:

`$ docker-compose up`

You might get permissions errors that docker is unable to mount that path. You will need to go to File Sharing in Docker Preferences and add your project\'s path.

Add a new theme to WordPress again and you must see the theme directory added to wp-content/themes directory in the local projects directory. If you restart the containers now the wp-content directory in local project directory will be mounted to the container and will show up in the themes page.

#### Summary

Installing the LAMP stack to run WordPress can be time-consuming and difficult to replicate between multiple systems. Docker containers make running MySQL and the WordPress server simple. Docker-compose simplifies managing configuration to run multiple docker containers. Data and content persistence can be achieved by using the volume mounts.

This project can be pushed to a git repository and it is easy to start development with minimal configuration. We will discuss the workflow for development and production deployment to a hosting provider in upcoming posts.
