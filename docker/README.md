# Docker Walk Through

If you haven't already, clone the repo

```
git clone https://github.com/bartr/aks-quickstart

cd aks-quickstart/docker
```

Create an Azure VM running Docker and Go

```
# this takes 5-10 minutes
./create


# if DHOST is empty, wait 30 seconds and run again until not empty
export DHOST=aks@`az network public-ip show -g aks -n dockerPublicIP --query [ipAddress] -o tsv`
echo $DHOST

# connect to the Docker VM you just created
ssh $DHOST

# Your prompt should look like this:
# aks@docker:~$

```

Run some docker commands

```
# run a web app
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
docker exec web pwd
docker exec web ls
docker exec web cat logs/app.log

# start an interactive shell in the container
docker exec -it web sh

# notice your prompt changed

# run a couple of commands and exit
pwd
ll
cat logs/app.log

exit
```

Stop and remove the web server

```
docker stop web
docker rm web
```

### Build a container


clone a sample Go app


```
git clone https://github.com/bartr/go-web-aks

# run the app locally
cd go-web-aks/app

# run in the background
go run main.go &

# make a web request
curl localhost:8080

# bring the web app to the foreground and stop it
fg
# press <ctl> c to stop the app


cd ..
```

## Build a docker container

```
docker build -t web .

# look at what happened
docker images

cat dockerfile

# run the container
docker run -d --name web -p 80:8080 web

# verify it's running
docker ps

# send a web request and look at logs
curl localhost
docker logs web

# stop and remove the web container
docker stop web
docker rm web

# should be nothing running
docker ps -a

# We're done!
exit

# Prompt should change
```

