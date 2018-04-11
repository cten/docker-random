# Meetup Docker Fayetteville April 10, 2018

## Farily common looking Dockerfile
### Dockerfile
[Dockerfile reference](https://docs.docker.com/engine/reference/builder/)
```dockerfile
FROM golang:1.9
WORKDIR /go/src
COPY goapp.go .
RUN go build -o /go/bin/goapp
EXPOSE 8080
ENTRYPOINT ["go/bin/goapp"]
```
### Build container image keeping it local
[Docker build reference](https://docs.docker.com/engine/reference/commandline/build/)
```bash
docker build -t goapp .
```
### Build container image to be pushed to remote registry, also using non-standard Dockerfile name (. = context)
```bash
docker build -t registry:5000/goapp:0.1 -f my-docker-file .
```
### Remove dangling images without prompt (untagged or intermediate images)
```bash
docker image prune -f
```
### Run newly built image with name 'testapp' as daemon exposing port 8080
```bash
docker run -d -p 8080:8080 -n testapp goapp
```
### Show all container instances, both running and stopped/crashed
```bash
docker ps -a
```
### Getting a shell inside the running instance
```bash
docker exec -it testapp sh
```
### Get verbose info on container
```bash
docker container inspect testapp
```
##
### Dockerfile for SCRATCH container using mutli-stage build
```dockerfile
FROM golang:1.9 AS build-env
WORKDIR /go/src/goapp
COPY goapp.go .
ENV CGO_ENABLED=0
ENV GOOS=linux
RUN go build -a -tags netgo -ldflags '-w' .

FROM scratch
EXPOSE 8080
COPY --from=build-env /go/src/goapp/goapp /
ENTRYPOINT ["/goapp"]
```
### Looking at another container's file system built from a SCRATCH image
```bash
docker build -t tiny .
docker run -d --name tiny1 tiny
docker run --rm -it --pid container:tiny1 alpine
```
Now you can look around by running commands like:
```bash
ps -ef
ls -la /proc/1/root/
```
### Get a shell inside a SCRATCH container
Great blog post on this stuff: https://embano1.github.io/post/scratch/
```bash
PID=$(docker inspect -f '{{.State.Pid}}' testapp)
cd /proc/$PID/root
```

## Volumes
[Dockerfile volume reference](https://docs.docker.com/engine/reference/builder/#volume)
[Docker run volume reference](https://docs.docker.com/engine/reference/run/#volume-shared-filesystems)
### Create dir on host with html, then use that dir as volume in container
```bash
mkdir -p ~/html-example && echo '<h1>Look I am here</h1>' > ~/html-example/index.html
docker run -d -p 80:80 -v ~/html-example:/usr/share/nginx/html nginx:alpine
```

#### GO App example code
```golang
package main

import (
    "fmt"
    "html"
    "log"
    "net/http"
)

func main() {

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
	    log.Printf("HOST: %s, METHOD: %s", r.Host, r.Method)
        fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
    })

    http.HandleFunc("/hi", func(w http.ResponseWriter, r *http.Request){
	    log.Printf("HOST: %s, METHOD: %s", r.Host, r.Method)
        fmt.Fprintf(w, "Hi")
    })

	log.Println("Listening on port 8080")
    log.Fatal(http.ListenAndServe(":8080", nil))

}
```