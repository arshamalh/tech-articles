---
title: "How docker will save you time, and why should you learn it immediately? (2/2)"
datePublished: Fri Mar 10 2023 05:18:28 GMT+0000 (Coordinated Universal Time)
cuid: clf237msm000b08l81kw6h6hz
slug: why-learn-docker-sooner-2
tags: docker, devops, dockerfile, docker-images

---

Hi guys! In the [last part](https://arshamalh.hashnode.dev/why-learn-docker-sooner-1), we discussed a little about what an image and a container are, in this part, we will make our own images and wrap our application in a container.

## Make a Dockerfile in the root

First of all, make a file named **Dockerfile**, without any extension, in the root directory:

![Dockerfile](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/56vcfv47map69jvm03p8.png align="left")

Tip: you can call this file whatever you want, but **Dockerfile** is the default recognizable name, other names, should be introduced in later steps.

## Starting our image

As I said before:

> Image is a collection of applications and commands running on a base image and this base image itself is a simplified Linux distro.

And a Dockerfile is an introduction file for our image. First of all, we should specify what base image we're going to run our application and commands on, and we do it like this:

```plaintext
FROM base_image
```

Alpine's latest version as base image:

```plaintext
FROM alpine
```

Alpine version (3.15) with a pre-installed node.js version (18):

```plaintext
FROM node:18-alpine3.15
```

And you can find other base images and their tags (versions) in the Dockerhub.

Tip: Always try to use tagged versions, even in the case of `latest`.

## Command the image!

You can make different users with different accessibilities in your image and run commands with their help, but for now, we're not going into that.

You're already familiar with **FROM** command that is used for pulling the base image, there are more:

---

**RUN**: The RUN instruction is used to run commands. From docker:

> Executes any commands on top of the current image as a new layer and commit the results.

Commands could be for example `npm install` or `go mod tidy` or `pip install -r requirements.txt` or more complex commands like `apt-get update && apt-get install -y curl` or any other command you run inside a Linux!

---

**CMD**: Provide defaults for an executing container, for example, `npm run start` or running a compiled executable.

Tip: there can only be one CMD instruction in a Dockerfile.

---

**COPY**: Copy files or folders from the root project directory (called source) to an image's filesystem path (called dest).

```plaintext
# COPY source dest
COPY ui .
COPY . ./
# Or many files, last argument is dest
COPY go.mod go.sum ./
# Or
COPY ["go.mod", "go.sum", "./"]
```

---

**WORKDIR**: The WORKDIR instruction sets the working directory for the next instructions that follow it in the Dockerfile.

```plaintext
FROM alpine
WORKDIR /app
# Next commands will run in /app
COPY file1 file2 ./
COPY file3 file4 ./
# now, we have /app/file1 /app/file2 etc.
CMD ["./main"]
# command above will look for /app/main
```

---

**ENV**: defines environment variables.

```plaintext
ENV PG_PASS=12@3fl
```

---

There are many other commands like **EXPOSE**, **ADD**, **ENTRYPOINT,** etc which you can read on the [Dockerfile reference](https://docs.docker.com/engine/reference/builder).

## Build images

Let's start with a node.js example, but don't worry if you're not a node.js developer.

After writing a complete Dockerfile like this:

```plaintext
FROM node:18-alpine3.15
COPY src .
COPY package*.json .
RUN npm install
CMD npm run start
```

Tip: package\*.json means all files starting with `package` and ending with `.json` =&gt; `package.json` and `package-lock.json`

We can build the image by running this command:

```bash
docker build <path>
```

We usually run this command inside the project root directory and use a single dot . for the path: `docker build .`

This command alone, assigns a random name for our container, but we can name it on our own using tags:

```bash
docker build -t <image_name> <path>
```

## Caching and layered images

The last version will work but is not performant, it's highly recommended to consider caching techniques.

Docker images are layered, simply said, every line in Dockerfile is a layer, and Dockerfile is read line by line.

What do we cache in the world of docker? Mostly, project dependencies and packages and also we do not COPY them from source to destination.

As you might know, `npm i` command is a so heavy command that installs project dependencies and reads dependency needs from the package.json file.

Look at the example below:  
project structure:

```plaintext
- index.js
- node_modules (dependencies)
- package.json
```

Dockerfile:

```plaintext
COPY index.js package.json ./
RUN npm install
CMD npm run start
```

It's good and working, but what happens if we change package.json and add new dependencies?

They will be installed and that's OK.

But what happens if we change index.js **without** changing dependencies? Dependencies will be installed again! and that's so NOT OK!

And that happens because we copy package.json and index.js at the same time, and we do the installation after that.

We should copy all files except dependency introducer (pakcage.json) after installing dependencies:

```plaintext
COPY package.json ./
RUN npm install
COPY index.js ./
CMD npm run start
```

It's way much faster!

in this case, if we change only index.js, docker will read Dockerfile from line 3 and go after that, because it's defined on line 3.

## Running your image (container)

Now we've learned how to make custom images, and we know how to run a container or expose ports from [the last tutorial](https://arshamalh.hashnode.dev/why-learn-docker-sooner-1).

Running your own container is the same as running a Redis or Postgres container. You can also make your own Redis or Postgres images!

## Exposing port

A container is an isolated environment, which means all ports are closed by default. If you have an HTTP server running inside a container, you should open (expose) your preferred port. There is an **EXPOSE** instruction, but I personally prefer using `-p <port>:<port>` when running a container in most cases, this actually overwrites **EXPOSE** as well.

## Resources and Final quote

I tried to help you understand that docker is not a challenge, it's a tool to make your life easier, this part covered the basics of writing a Dockerfile and how to make your custom images.

There are so many tutorials out there, but I'm gonna suggest the ones that I enjoyed the most. As an instruction that shows how containerization will help us:  
[Containerization explained](https://www.youtube.com/watch?v=0qotVMX-J5s&list=PLOspHqNVtKABAVX4azqPIu6UfsPzSu2YN&index=2)  
[Docker and Kubernetes course by Stephan Grider](https://www.udemy.com/course/docker-and-kubernetes-the-complete-guide)