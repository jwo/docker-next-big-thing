##Docker - next big thing

### By @jwo for HoustonRB

---

##Docker

-	Run in linux, just like production
-	It's like heroku, for development

![](http://www.rudebaguette.com/assets/docker.jpg)

---

##Laws of Docker

-	1 Process per Container
-	"Containers" are servers

---

##Standard Rails App

-	Rails
-	PostgreSQL
-	Redis
-	Memcached
-	Sidekiq

---

##Startup - Development

-	Clone
-	bundle
-	rake db:setup
-	brew install postgresql
-	createdb
-	brew install redis
-	brew install memcached
-	foreman start

---

##Startup - Production

-	install ruby from source. maybe rbenv?
-	sudo apt-get install postgresql, redis, memcached
-	bundle
-	create database and user
-	rake db:migrate
-	configure nginx and unicorn

---

##ETOOMANYDIFFS

####New users, many dependencies, such frustration

![](http://media0.giphy.com/media/GDnomdqpSHlIs/giphy.gif)

---

##Startup, Dev and Live

1.	`git clone`
2.	`docker-compose build`
3.	`docker-compose up`

---

##Comparison vs Chef

-	No need for chef or ansible for most cases
-	Deploy and you're good

---

##Docker Compose

-	Was "fig" (older documentation)
-	Fancy term: "Coordination"
-	Real World: connects your DB to your Rails

---

##docker-compose.yml

```
mysqldata:
  image: cogniteev/echo
  volumes:
    - /var/lib/mysql

db:
  build: mysql:latest
  expose:
    - "3306"
  volumes_from:
    - mysqldata

app:
  build: .
  command: bundle exec rails s -p 3000 -b '0.0.0.0'
  volumes:
    - .:/myapp
  ports:
    - "3000:3000"
  links:
    - db
```

---

##Boot2docker

Have to have it so your Macs can run Linux in your Linux

![](https://media1.giphy.com/media/ToMjGpM4h8DAsgL0fEA/200.gif)

---

##HOWTO Persist Data

(Like Postgres, etc Data)

-	Have your db data directory connect to a "data container"

```
mysqldata:
  image: cogniteev/echo
  volumes:
    - /var/lib/mysql

db:
  build: mysql:latest
  expose:
    - "3306"
  volumes_from:
    - mysqldata
```

---

##HOWTO Deploy your docker

-	SSH into your main box
-	`git clone`
-	`docker-compose build`
-	`docker-compose stop && docker-compose start`

---

##HOWTO Get a main box with docker

1.	Digital Ocean $5 plan
2.	Choose Docker
3.	Win.

---

##HOWTO Edit your Rails locally

-	Do everything the same except: http://192.168.59.103:3000/ instead of http://localhost:3000

---

##HOWTO run migrations

```
docker-compose run app rake db:migrate
```

---

##HOWTO Environment files

Add in a .env file like you normally would. Gitignore it.

```
mysql:
  env_file: DockerConfig/.env
  volumes_from:
    - mysqldatacontainer
```

---

##HOWTO database.yml

```
production:
  adapter: mysql2
  database: yolo_production
  username: <%= ENV['MYSQL_USER']%>
  password: <%= ENV['MYSQL_PASS']%>
  host: mysql
  port: 3306
```

---

###HOWTO SSH into your boxes

1.	Don't
2.	Instead, run a process like you would heroku

```
docker-compose run app /bin/bash
```

---

###HOWTO Monitor your boxes

Prometheus, by SoundCloud folk, rules.

```
prometheus:
  build: docker_config/prometheus
  ports:
    - "9090:9090"
  links:
    - containerexporter

containerexporter:
  image: prom/container-exporter
  ports:
    - "9104:9104"
  volumes:
    - /sys/fs/cgroup:/cgroup
    - /var/run/docker.sock:/var/run/docker.sock
```

---

### DockerFile

```
FROM phusion/passenger-ruby22
RUN apt-get update -qq && apt-get install -y build-essential

RUN mkdir /myapp
WORKDIR /myapp
ADD . /myapp
RUN bundle install
```

---

### Services

-	Docker intentionally breaks upstart and most things that start systems
-	The Dockerfile, instead, has a "CMD" (one cmd) that runs when started
-	This is basically the docker way

---

###HOWTO Run more than one service

(assuming you want to say, run Inspeqtor to notify you of crashes)

-	Install runit (pronounced 'run-it')
-	CMD is "/usr/sbin/runsvdir-start"

```
# Add runit startups
RUN mkdir /etc/sv/inspeqtor
ADD inspeqtor.runit /etc/sv/inspeqtor/run
RUN mkdir /etc/sv/mysql
ADD mysql.runit /etc/sv/mysql/run
RUN chmod +x /etc/sv/inspeqtor/run
RUN chmod +x /etc/sv/mysql/run
# Link runit startup scripts
RUN ln -s /etc/sv/mysql/ /etc/service/mysql
RUN ln -s /etc/sv/inspeqtor/ /etc/service/inspeqtor

# remove the upstart config directory
RUN rm -rf /etc/init
```

---

![](http://media.giphy.com/media/mGWWxN49a6jwA/giphy.gif)

---

###Yeah

-	So that part is rough around the edges
-	Docker REALLY wants you to do 1 process per container
-	It strips all else away besides what you start
-	But runit can work

---

###Finding Images

Images are on https://registry.hub.docker.com/

---

###HOWTO Reverse Engineer Dockerfiles

1.	Check out the Image Dockerfile
2.	Copy and Paste
3.	Make sure you're on same linux OS.

Ubuntu. CoreOS. Debian. (likely more).

---

###JWO Rule of Docker Images

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">What if I told you that using other people&#39;s (even official) docker images lead to more problems than they solve?</p>&mdash; Jesse Wolgamott (@jwo) <a href="https://twitter.com/jwo/status/592726544486903808">April 27, 2015</a></blockquote><script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

---

###HOWTO Scale number of app servers

1.	`docker-compose scale app=2`
2.	Use NGINX as reverse proxy
3.	Have ngnix server port 80, server to many upstream

```
upstream app_servers {
  server 127.0.0.1:8080;
  server 127.0.0.1:8081;
}
```

Possible way to automate: http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/

---

###HOWTO Have a larger disk size on boot2docker

```
boot2docker destroy
boot2docker init --disksize=20000
```

(Yeah, that deletes all stuff and you start over)

---

###HOWTO See what's running, and stop them

` docker ps `

```
CONTAINER ID        IMAGE                    COMMAND                CREATED STATUS              PORTS               NAMES
555b3b4e31fa        fivehundred_app:latest   "bundle exec rails s   2 minutes ago       Up 2 minutes        80/tcp, 443/tcp     fivehundred_app_1   
6a5bb766f745        postgres:latest          "/docker-entrypoint.   2 minutes ago       Up 2 minutes        5432/tcp            fivehundred_db_1    
```

`docker kill 555b3b4e31fa`



---

###Demo

---

###Conclusion

1.	Docker is extremely cool
2.	Fast, easy'ish (appropriate levels of easy)

#####Try it out:

[GitHub / jwo / blocker](https://github.com/jwo/blocker)
