# laravel-docker-template

[https://dev.to/ahmedash95/simple-steps-to-dockrize-your-laravel-app-4h40](https://dev.to/ahmedash95/simple-steps-to-dockrize-your-laravel-app-4h40)

Hello Friends, I want to share something that I used to do few months ago for my laravel projects. 

The reason I decided to drop Valet is that I'm using a shared Macbook pro that has the company dev environment and the projects I'm working on every day after work.

Valet was not quite a good option as it somehow conflicts with the tools I use for my company dev setup. so I decided to dockerize everything to make it easy for me to work and even share projects with other friends if we work on it together.

here I share how I set up every project I start.feel free to share your thoughts, ideas, comments and feedback on it. also, I know it's not that fast to get it for every project. but as a template, it does a good job for the work.

### Prerequisite

The setup I use can be for new projects or existing projects. tools used is docker and other project dependencies like MySQL, Redis, NPM for frontend development. 

I'll start everything with an empty directory and you can jump between steps. also, I assume you already have docker and docker-compose on your machine and you can read docker-compose.yaml and Dockerfiles

### Preparing the files

I'll start with 4 images 

- Nginx: as a web server
- PHP: contains php-fpm, php-cli, composer. I even install npm there to easy run compile laravel assets instead of installing another image
- MySQL: the database we need for our app
- Redis: for caching, queues

```bash
    .
    ├── docker
    │   ├── nginx
    │   │   └── conf.d
    │   │       └── default.conf
    │   └── php
    │       └── Dockerfile
    └── docker-compose.yml
    
    4 directories, 3 files
```

We have **docker-compose.yml** and docker directory that has the configuration files. and for php we have a custom image

# The docker-compose

We will define some volumes to copy important configurations and to keep data in sync between our machine and the containers so even with a rebuild or restart containers the data still safe and not destroyed.

*Check the comments in the file to know more*

```yaml
    version: '2'
    services:
      webserver:
        image: nginx:alpine
        volumes:
          - ./code:/var/www
    			## copy nginx configuration for our application ##
          - ./docker/nginx/conf.d/:/etc/nginx/conf.d/
        ports:
    			## run the webserver on port 8080 ##
          - "8080:80"
      app:
    		## read php image from our custom docker image ##
        build:
          context: .
          dockerfile: ./docker/php/Dockerfile
        volumes:
          ## copy project files to /var/www ##
          - ./code:/var/www
        working_dir: /var/www
    
      db:
        image: mysql:5.7
        ## expose the mysql port to our machine so we can access it from any mysql-client like TablePlus ##
        ports:
          - "3388:3306"
       ## keep mysql data on localhost so we don't lose them ##
        volumes:
          - ./docker-volumes-data/db:/var/lib/database
        environment:
          MYSQL_DATABASE: laravel_db
          MYSQL_ROOT_PASSWORD: root
      redis:
        image: redis
        volumes:
          ## keep redis data on localhost so we don't lose them ##
          - ./docker-volumes-data/redis:/data
```

Now let's jump into each one and check it's content

### Nginx: webserver

```yaml
    webserver:
        image: nginx:alpine
        volumes:
          - ./code:/var/www
    			## copy nginx configuration for our application ##
          - ./docker/nginx/conf.d/:/etc/nginx/conf.d/
```

here we copy the project files and also copy all conf.d files. which has one file called default.conf and the file has the virtual host for our project
```nginx
    server {
    	client_max_body_size 30M;
        listen 80;
        index index.php index.html;
        error_log  /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log;
        root /var/www/public;
        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass app:9000;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
        }
        location / {
            try_files $uri $uri/ /index.php?$query_string;
            gzip_static on;
        }
    }
```

### PHP
```yaml
    app:
        ## read php image from our custom docker image ##
        build:
          context: .
          dockerfile: ./docker/php/Dockerfile
        volumes:
          ## copy project files to /var/www ##
          - ./code:/var/www
        working_dir: /var/www
```
for php we have a custom dockerfile for it so we can install the extensions we need and customize the setup for our need. the file fits any project I work on so it might fit your need to
```docker
    # dockerfile
    FROM php:7.2-fpm
    
    # Copy composer.lock and composer.json
    COPY code/composer.lock* code/composer.json* /var/www/
    
    # Set working directory
    WORKDIR /var/www
    
    # Install dependencies
    RUN apt-get update && apt-get install -y \
        build-essential \
        libpng-dev \
        libjpeg62-turbo-dev \
        libfreetype6-dev \
        locales \
        zip \
        jpegoptim optipng pngquant gifsicle \
        unzip \
        curl
    
    # Clear cache
    RUN apt-get clean && rm -rf /var/lib/apt/lists/*
    
    # Install extensions
    RUN docker-php-ext-install pdo pdo_mysql mbstring zip exif pcntl
    RUN docker-php-ext-configure gd --with-gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ --with-png-dir=/usr/include/
    RUN docker-php-ext-install gd
    RUN pecl install -o -f redis \
        &&  rm -rf /tmp/pear \
        &&  docker-php-ext-enable redis
    
    # Install composer
    RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    
    # NPM for frontend builds
    RUN apt install nodejs npm -y
    
    # Add user for laravel application
    RUN groupadd -g 1000 www
    RUN useradd -u 1000 -ms /bin/bash -g www www
    
    # Copy existing application directory contents
    COPY ./code /var/www
    
    # Copy existing application directory permissions
    COPY --chown=www:www . /var/www
    
    # Change current user to www
    USER www
    
    # Expose port 9000 and start php-fpm server
    EXPOSE 9000
    CMD ["php-fpm"]
```
here I use php7.2 and then I add some important extensions with composer and NodeJS and NPM to compile the assets from the same image. of course, you can move that part to another NodeJS image but I prefer to keep it there as it works perfectly for me.

### MySQL

mysql has a very easy setup
```yaml
    db:
        image: mysql:5.7
        ports:
          - "3388:3306"
        volumes:
          - ./docker-volumes-data/db:/var/lib/database
        environment:
          MYSQL_DATABASE: laravel_db
          MYSQL_ROOT_PASSWORD: root
```
I expose the port to `3388` so I can connect to it using any mysql-client like TablePlus or SequelPro 

<img src="https://res.cloudinary.com/ahmedash/image/upload/v1576622303/dev.to/dockerize-laravel/Untitled.png" />

Also, I use `volumes` to sync the data between the host and the container so I don't lose the data if it restarts or stops 

### Redis

```yaml
    redis:
        image: redis
        volumes:
          - ./docker-volumes-data/redis:/data
```

I use `volumes` to sync the data too but as you see I don't expose any ports as I use the terminal to check Redis. 

laravel `queue:listen` command is enough for me to monitor the jobs during development

# Run docker build and up

Now it's time to start our docker setup and then initialize our project 
```bash
    $ docker-compose up --build
```
I run this command every time to build the PHP docker image and run the containers. it will take some time to install everything and the end result should be like:
```
    Successfully built b98842b1e665
    Successfully tagged docker-demo_app:latest
    Creating docker-demo_db_1        ... done
    Creating docker-demo_app_1       ... done
    Creating docker-demo_webserver_1 ... done
    Creating docker-demo_redis_1     ... done
```
Now we have every thing ready let's move to the next step

### Initialize the project

If you are running a new laravel project then we will run the composer from the app docker container to install the new project. first, we need to get the container name. you can get it from the output of the previous command or just run 
```bash
    $ docker-compose ps
    
    # output #
    
             Name                        Command               State                 Ports
    ----------------------------------------------------------------------------------------------------
    docker-demo_app_1         docker-php-entrypoint php-fpm    Up      0.0.0.0:2222->22/tcp, 9000/tcp
    docker-demo_db_1          docker-entrypoint.sh mysqld      Up      0.0.0.0:3388->3306/tcp, 33060/tcp
    docker-demo_redis_1       docker-entrypoint.sh redis ...   Up      6379/tcp
    docker-demo_webserver_1   nginx -g daemon off;             Up      0.0.0.0:8080->80/tcp
```
Then let's run 
```bash
    $ docker exec -it docker-demo_app_1 composer create-project laravel/laravel code
```
it'll install laravel project inside `/code` directory

### Basic laravel stuff

we need to run few commands to get the app ready

### Key generate
```bash
    $ docker exec -it docker-demo_app_1 php artisan key:generate
    
    # output #
    Application key set successfully.
```
### Modify .env file

we just need to change the `database` and `redis` to make them work for our app
```
    # database config
    DB_CONNECTION=mysql
    DB_HOST=db
    DB_PORT=3306
    DB_DATABASE=laravel_db
    DB_USERNAME=root
    DB_PASSWORD=root
    
    # redis config
    REDIS_HOST=redis
    REDIS_PASSWORD=null
    REDIS_PORT=6379
```
Let's run the migration files to make sure the database works well
```bash
    $ docker exec -it docker-demo_app_1 php artisan migrate
    
    # output #
    Migration table created successfully.
    Migrating: 2014_10_12_000000_create_users_table
    Migrated:  2014_10_12_000000_create_users_table (0.05 seconds)
    Migrating: 2014_10_12_100000_create_password_resets_table
    Migrated:  2014_10_12_100000_create_password_resets_table (0.02 seconds)
    Migrating: 2019_08_19_000000_create_failed_jobs_table
    Migrated:  2019_08_19_000000_create_failed_jobs_table (0.01 seconds)
```
### Run tests
```bash
    $ docker exec -it docker-demo_app_1 ./vendor/bin/phpunit
    PHPUnit 8.5.0 by Sebastian Bergmann and contributors.

    ..                                                                  2 / 2 (100%)
    
    Time: 2.1 seconds, Memory: 16.00 MB
    
    OK (2 tests, 2 assertions)
```
### Run on browser

just visit [http://localhost:8080/](http://localhost:8080/) to see the app running

# Source code

here are the full files on Github where you can clone and use them

[https://github.com/ahmedash95/laravel-docker-template](https://github.com/ahmedash95/laravel-docker-template)

# Conclusions

I hope the article helped you to easily create a docker setup for your app. even make it work with the existing apps shouldn't be a problem at all.

I still see it's not smooth enough. but I hope to find a better way to make it simpler. you can help by sharing ideas or any experience you had with a similar case.

I know there is [laradock](https://laradock.io/) but I feel it's too big to use it and just have 4 files make it easy to get it running.
