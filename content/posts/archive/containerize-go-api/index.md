---
title: "Containerizing a Go API with Docker For Mac"
date: 2018-04-06T07:10:00-04:00
description: Containerizing a Go API with Docker For Mac
menu:
  sidebar:
    name: Containerizing a Go API with Docker For Mac
    identifier: containerize-go-api
    parent: archive
    weight: 300
---

I’m working through creating a [RESTful API template](https://github.com/gilcrest/go-API-template). As part of it, I want to be able to “Containerize” my app using docker and deploy it to “the cloud”. Baby steps for me though — I want to get everything working locally first. This post is about “containerizing” my API using Docker and getting it to work locally on my Mac. Right now, from a networking perspective, my app is pretty simple — it needs connectivity on two ports: 1 port for database traffic, 1 port for http traffic.

## Part 1 — Simple build against local PostgreSQL, exposing ports 5432 for PostgreSQL and 8080 for http traffic from my local host to my container

First off, you must have Docker for Mac installed. Once installed, you’ll need to familiarize yourself with the various docker commands and idioms. I highly recommend [Docker Deep Dive](https://smile.amazon.com/Docker-Deep-Dive-Nigel-Poulton-ebook/dp/B01LXWQUFF/ref=mt_kindle?_encoding=UTF8&me=). I read a lot of different blogs and how-tos on Docker and wasn’t able to really put it all together until I read this book.

For my database, I have PostgreSQL running locally on my Mac using [Postgres.app](https://postgresapp.com/) — you can use whatever you like, but I find this product incredibly simple to use. I do not change any of the default settings, which means Postgres listens on port 5432.

For database configuration (dbname, host, port, user, password), I’m storing everything as [environment variables](https://www.digitalocean.com/community/tutorials/how-to-read-and-set-environmental-and-shell-variables-on-a-linux-vps) and accessing them within my Go program via os.Getenv, e.g.:

```go
dbName := os.Getenv("PG_DBNAME_TEST")
dbUser := os.Getenv("PG_USERNAME_TEST")
dbPassword := os.Getenv("PG_PASSWORD_TEST")
dbHost := os.Getenv("PG_HOST_TEST")
dbPort, err := strconv.Atoi(os.Getenv("PG_PORT_TEST"))
if err != nil {
    log.Error().Err(err).Msg("Unable to complete string to int conversion for dbPort")
    return nil, err
}
```

My Dockerfile for this part is quite simple and similar to the one on the [official golang blog](https://blog.golang.org/docker). I use dep (will eventually switch to vgo) to “vendor my dependencies” so as part of my build I don’t need to “go get” any external dependencies as they’re copied over as part of the ADD command when I copy the entire workspace over from my local host to the container.

```dockerfile
# Start from a Debian image with the latest version of Go installed
# and a workspace (GOPATH) configured at /go.
FROM golang:latest

# Create WORKDIR (working directory) for app
WORKDIR /go/src/github.com/gilcrest/go-API-template

# Copy the local package files to the container's workspace
# (in the above WORKDIR)
ADD . .

# Switch WORKDIR to directory where server main.go lives
WORKDIR /go/src/github.com/gilcrest/go-API-template/cmd/server

# Build the go-API-template userServer command inside the container
# at the most recent WORKDIR
RUN go build -o userServer

# Run the userServer command by default when the container starts.
# runs command at most recent WORKDIR
ENTRYPOINT ./userServer

# Document that the container uses port 8080
EXPOSE 8080

# Document that the container uses port 5432
EXPOSE 5432
```

Let’s break down what’s happening in this file:

**FROM golang:latest** — This command tells Docker to pull the golang image with the latest tag from the official Docker repository as the first layer.

**WORKDIR /go/src/github.com/gilcrest/go-API-template** — This command sets the working directory to the path given. This directory doesn’t already exist, so, it will be created.

**ADD . .** — This command copies all the files, directories and subdirectories from the current directory where the Dockerfile is located and moves them to the WORKDIR defined above

**WORKDIR /go/src/github.com/gilcrest/go-API-template/cmd/server** — This command sets the working directory to the path given. This directory exists already.

**RUN go build -o userServer** — This command builds the application binary from main.go within the work directory and creates the binary with the given name userServer

**ENTRYPOINT ./userServer** — This command sets the container to run as an executable and runs the userServer binary created earlier

**EXPOSE 8080 and EXPOSE 5432** — These commands are really documentation only. They document that this image needs these ports exposed to function properly.

To build the image from my Dockerfile, I run the following command from my app’s root directory (which is where I also store my Dockerfile):

```bash
# Build image with gilcrest as repository name, go-api-template 
# as build name and latest as build tag from the current directory

$ docker image build -t gilcrest/go-api-template:latest .
```

After successfully building my new image, I use the below command to run it:

```bash
# Run container using previously built image (gilcrest/go-api-template)
# -d Run in the background (detached)
# -p publish port 5432 on the host to port 5432 on the container for postgres
# -p publish port 8080 on the host to port 8080 on the container for http access
# --env-file load environment variables using the env file
# --name name the container user-server

$ docker container run -d -p 5432:5432 -p 8080:8080 --env-file ./test.env --name user-server gilcrest/go-api-template
```

To quickly go through all the options as part of the run command above:

**-d**: Runs the container in the background as a “detached” session or daemon

**-p**: publishes ports from the host to ports on the container (host on left of colon, container on right of colon)

**-—env-file**: where to pull “environment file” for setting environment variables at run time

**-—name**: gives the container a unique identifier

---

A couple of gotchas I’d like to point out that I ended up spending hours researching that may be helpful for some.

To connect to PostgreSQL running on your Mac (the docker “host”) from within a running container, you have to know the host’s IP address. Seeing as when running a container as part of Docker for Mac, you’re actually running within a lightweight VM, determining your actual host IP address can be a tricky proposition as the IP is not static. Luckily, more recent Docker for Mac versions allow you to use special DNS name host.docker.internal to determine the host IP address. See [I WANT TO CONNECT FROM A CONTAINER TO A SERVICE ON THE HOST](https://docs.docker.com/docker-for-mac/networking/#use-cases-and-workarounds) on the docker site for the full details.

Another challenge was actually connecting to my API endpoint running in my container from my local machine. Prior to using Docker, I had been specifying a particular IP address for my localhost when setting up http.ListenAndServe, e.g. **http.ListenAndServe(“127.0.0.1:8080”, nil)**, but seeing as when you run your container using Docker for Mac, you’re running within a lightweight VM, it will have a different IP than the standard loopback localhost IP (127.0.0.1). I tried docker inspect but had a pretty hard time figuring out exactly what IP I would need to try and hit to make this work. I ended up removing the loopback localhost IP from the addr function parameter for http.ListenAndServe and only have the port as part of the string, as in below (“:8080”).

```go
// ListenAndServe on port 8080, not specifying a particular IP address for this particular implementation
if err := http.ListenAndServe(":8080", nil); err != nil {
    log.Fatal(err)
}
```

Once you’ve removed this IP, you can then use “localhost” and port 8080 to connect to your APIs running within the container. Requests will be properly forwarded (e.g. a POST to[http://localhost:8080/api/appUser](http://localhost:8080/api/appUser) now works and hits my container API!) See [I WANT TO CONNECT TO A CONTAINER FROM THE MAC](https://docs.docker.com/docker-for-mac/networking/#use-cases-and-workarounds) on the Docker site for full details.

One thing to highlight from the former run command is the environment file that I use to load the environment key:value pairs from my local shell to the container’s shell environment.

```bash
$ docker container run -d -p 5432:5432 -p 8080:8080 --env-file ./test.env --name user-server gilcrest/go-api-template
```

This file, which simply looks like:

```bash
PG_DBNAME_TEST
PG_USERNAME_TEST
PG_PASSWORD_TEST
PG_HOST_TEST=host.docker.internal
PG_PORT_TEST
```

— tells Docker to pull in the above environment variables from the local shell and push them into the containers shell. For most of the variables, I am only providing the key and not the value. Docker will pull in the value from the shell environment that is running the command. For the PG_HOST_TEST variable I am providing the value (host.docker.internal) as I want to override the shell environment value on my Mac (localhost). This allows me to run my app either inside a container or just run it locally without a container as you would any non-containerized app, and it works in both cases. If I am running the app “normally” (non-containerized), the environment will have ***localhost*** as PG_HOST_TEST, if I’m running inside the container, I need ***host.docker.internal*** as the host name for the db.

Storing my configuration in the environment like this meets the “Twelve Factor App” guidelines, which I think are generally good. I make sure I have .env files in my .gitignore file to ensure that these config values don’t get checked in as part of my app (see below for .gitignore example).

```bash
# Docker env files
*.env
```

---

Now that we’ve got the basics working, the image that is created is really pretty big — for my app it’s 882mb! If you’re building for the first time, then because of this size, the build process is also pretty slow…

As Gophers, naturally, we like our builds fast. In order to run smaller, faster, more secure/efficient images, Docker allows for [multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/). As far as I can tell, this is pretty commonplace for running production containers and is considered a best practice.

## Part 2 — Dockerizing your app using Multi-Stage Builds

Docker allows for “multi-stage builds”, meaning you can pull down multiple images and copy artifacts from one stage to another — you can read all about it [here](https://docs.docker.com/develop/develop-images/multistage-build/). After trolling Slack and other blogs for information, the most common way I could find was to deploy apps using the Docker [Alpine Linux image](https://hub.docker.com/_/alpine/). This is a tiny image — 5mb!!! This makes building a lot faster and images way smaller. There is also an official golang Docker version using Alpine — [golang:alpine](https://hub.docker.com/_/golang/). This image isn’t as bare bones as the base Alpine Linux image as it has what’s needed to run Go.

I found several different methods of running containers using Alpine, many of which started with a “normal” Go base image, but that also requires you to compile the go binary in a special way as the Alpine Linux image is based on musl libc and busybox. My understanding is that because of this, you need to run a command similar to `RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .` to build your app. I don’t really know exactly what this command is doing, so I looked elsewhere.

An alternative is to use the golang:alpine image as your base image in your first “builder” stage and build as you normally would within it (e.g., go build -o app). As this image is already built within alpine, there seems to be no need to do the more complicated build command I mentioned above. You can then copy your app binary built within the first stage into your second stage which uses the alpine base image and it will run! My Dockerfile example is below:

```dockerfile
####################################################################
# Builder Stage                                                    #
####################################################################
# Use the official Golang image to create a build artifact.
# This is based on Debian and sets the GOPATH to /go.
# https://hub.docker.com/_/golang
FROM golang:alpine AS builder

# Create WORKDIR using project's root directory
WORKDIR /go/src/github.com/gilcrest/go-API-template

# Copy the local package files to the container's workspace
# in the above created WORKDIR
ADD . .

# Build the go-API-template command inside the container
RUN cd cmd/server && go build -o userServer

####################################################################
# Final Stage                                                      #
####################################################################
# Use the official Alpine image for a lean production container.
# https://hub.docker.com/_/alpine
# https://docs.docker.com/develop/develop-images/multistage-build/#use-multi-stage-builds
FROM alpine:3

# Create WORKDIR
WORKDIR /src/github.com/gilcrest/go-API-template/input

# Copy json file needed for feature flags to directory expected by app
# File is copied from the Builder stage image
COPY --from=builder /go/src/github.com/gilcrest/go-API-template/input/httpLogOpt.json .

# Create WORKDIR
WORKDIR /app

# Copy app binary from the Builder stage image
COPY --from=builder /go/src/github.com/gilcrest/go-API-template/cmd/server/userServer .

# Run the userServer command by default when the container starts.
ENTRYPOINT ./userServer

# Document that the service uses port 8080
EXPOSE 8080

# Document that the service uses port 5432
EXPOSE 5432
```

I went through most of the commands before, so I’ll just highlight a few things:

**FROM** — There are two FROM commands — this is the essence of multi-stage builds. You can have as many FROM commands (stages) as you want… NOTE!! I have given the first stage a name using AS (FROM golang:alpine AS builder). This is important for later…

**RUN cd cmd/server && go build -o userServer** — This command is two commands in one (the && allows for this). I first change the directory to cmd/server (remember, my working directory is the WORKDIR so I can work relative to that after the WORKDIR command). I then run the go build command sending it’s output binary to use userServer as it’s name.

**COPY** — from=builder /go/src/github.com/gilcrest/go-API-template/cmd/server/userServer . — The COPY command is used to copy files or directories from a source to a destination within the container. In this case, I’m copying from the builder stage I had name earlier to the WORKDIR I’m working in — cool! I use the COPY command to copy a json file my app needs as well as my app binary.

All the other instructions are pretty straightforward and work the same as I had mentioned in Part 1, including the docker build and docker run commands. The only gotcha I’ve found so far is that if you for some reason want to run your container in interactive mode using `-it` (instead of `-d`) and have it run using `/bin/bash` as you normally would, that won’t work on an alpine container as it’s not there! You have to use `/bin/sh` instead.

That’s it for today — my next post will be about getting this container hoisted up to “the cloud” somehow… Onwards and upwards!
