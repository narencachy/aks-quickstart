# Docker Walk Through

If you haven't already, clone the repo and run setup as explained in the [readme](README.md)

# connect to the Docker VM you just created

```
# to connect to the Docker build server
export DHOST=aks@`az network public-ip show -g $AKSRG -n dockerPublicIP --query [ipAddress] -o tsv`
ssh $DHOST
```

Your prompt should look like this:

aks@docker:~$

### Run some docker commands

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
ls
cat logs/app.log

exit
```

### Stop and remove the web server

```
docker stop web
docker rm web
```

## Build a container

### Clone a sample Go app

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

### Build a docker container

```
docker build -t web .

# look at what happened
docker images

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

exit
```

## We're done!
