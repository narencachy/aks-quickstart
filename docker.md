# Docker Walk Through
### 100 level

If you haven't already, clone the repo and run setup as explained in the [readme](README.md)

### Connect to your build server

Follow the steps in [readme](README.md) to connect to your build server via SSH

Your prompt should look like this:

aks@docker:~/$

### Make sure post install script has completed

```

# should return ready
cat status

```

### Some basic docker commands

```

# show local images
docker images

# pull an image
docker pull alpine
docker images

# let's run the image interactively
docker run -it --name ubu ubuntu

# notice your prompt changed to something like this: root@257fde9a1ad2:/#
# we are now "in" the docker container
pwd
cd root

ping www.microsoft.com

# oops - ping isn't installed
# let's install ping, curl and redis cli
apt-get update
apt-get install -y iputils-ping curl redis-tools

ping -c 3 www.microsoft.com
curl www.microsoft.com

exit

```

### We don't want to have to do that every time ...

```

# save our changes to a new image
docker commit ubu ubu
docker images

# let's run our new image
docker run -it --name ubu ubu

# oops
docker ps -a

# we have to remove the instance first
docker rm ubu
docker run -it --name ubu ubu

# your prompt changes again
# we're in the root directory (/) and we want to start in the home directory (~)
exit

# tell docker where to start
docker commit -c "WORKDIR /root" ubu ubu

# remove the instance and run again
docker rm ubu
docker run -it --name ubu ubu

# there's a MUCH better way! We'll get there soon.

```


### Run a simple web app in Docker

```

# run a simple web app
docker run -d -p 80:8080 --name web bartr/go-web-aks

# see what happened
docker images
docker ps
docker logs web

# send a request to the web server
curl localhost

# recheck the logs
docker logs web

# run some commands in the container
docker exec web ls -al
docker exec web cat logs/app.log

# start an interactive shell in the container
docker exec -it web sh

# notice your prompt changed

# run a couple of commands and exit
ls -al
cat logs/app.log

exit

```

### Remove the web container

```

docker rm web

# oops - the container is still running, so we need to stop it first
docker stop web

# we can restart it
docker start web
curl localhost

# stop and remove
docker stop web
docker rm web

# you could do this instead
docker rm -f web

```

## Build a container

At the end of the first section, we said there was a better way ...

### Clone a sample Go app

```

git clone https://github.com/bartr/go-web-aks
cd go-web-aks

```

### Build the docker container

```

docker build -t web .

# look at what happened
docker images

# let's see what we told Docker to do
# notice these commands are very similar to the first section
cat dockerfile

```

### Run the container

```

docker run -d --name web -p 80:8080 web

# verify it's running
docker ps

# send a web request and look at logs
curl localhost
docker logs web

```

### Stop and remove the web container

```

docker stop web
docker rm web

# should be nothing running
docker ps -a

```

### Let's run something a little more complex ...
```

# run a Redis container
docker run -d --name redis redis

# restart the ubu container
docker start -ai ubu

ping redis

# oops
# We have to attach the container to the same network
exit

# create the network
docker network create vote

# add the containers to the network
docker network  connect vote redis
docker network  connect vote ubu

# let's try again
docker start -ai ubu

ping redis

# w00t
exit

# let's run a web app that talks to the redis cache
# notice we attach it to the network
docker run -d --net vote --name govote bartr/govote

docker start -ai ubu

curl govote:8080

# sweet
exit

```

### What if we want to curl the website from our dev server?

```
docker rm -f govote

# let's expose Redis too
docker rm -f redis

# same run command with -p
docker run -d --name redis --net vote -p 6379:6379 redis

# let's see if it works
bin/redis-cli

set Dogs 100
set Cats 1
exit

# Same run command with -p
docker run -d --net vote --name govote -p 80:8080 bartr/govote

curl localhost

# Dogs RULE!
```

### Container size matters

Notice the size difference in the diferent images - from 12 MB to 700 MB

The size of the image has a big impact on deployment time, so you want to minimize your image size. That typically also reduces your security footprint. In the walkthrough, we used bad docker build practices but you have to start somewhere ...

```

docker images

```

### Cleanup

```

# if you don't remove these, parts of the AKS walk through will break as the ports will be in use
docker rm -f govote
docker rm -f redis
docker rm ubu

```

## We're done!
