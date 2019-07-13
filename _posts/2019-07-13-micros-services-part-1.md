---
layout: posta
title: Hosting a .NET Core Web API service in a docker container
excerpt: Creating a simple .NET Core Web API service and running it in a Docker container...
description: what
platform: MacOS High Sierra
reqs: Docker, Visual Studio Code
tags: docker web-api
---

* auto-gen TOC:
{:toc}

# Creating a Web API service
First we create a new project in Visual Studio Code.
* open Visual Studio Code
* switch to the integrated terminal
* navigate to the directory in which we want to create the project folder, in my case this is - spoiler alert - microservices
```shell
cd microservices
```
* and create a new Web API project using the following command, where `identity` is the name of my project.
```shell
dotnet new webapi -o identity
```
* to open the newly created project in Visual Studio Code we can use
```shell
code -r identity
```

And that's basically it. The newly created project contains one sample end point and that's all we need to proceed to Docker.

But before that, let's run the project to make sure everything works as expected. Go to Debug > Start Without Debugging or Press <kbd>Ctrl</kbd> + <kbd>F5</kbd> and a browser window should open.
Add `/api/values` to the URL in the address bar and hit <kbd>Enter</kbd>. We should now see a JSON response that looks like this:
```json
["value1","value2"]
```

# Running the Web API service in a Docker container
## What is Docker?
Docker (Engine) is a software that hosts containers. So then, what are containers? Simplified, you can think of them as virtual machines without an operating system. So instead of a host OS running a virtual machine with its own guest OS, a docker container shares the OS kernel with its host OS. One benefit of this approach is:
* no guest OS means less overhead and therefore less ressource consumption (less disk space, less RAM) and faster 'boot-time'

a disadvantage is:
* this only works if the software in our container can run on our host's OS. So running Windows containers on a Linux host won't work.

## Dockerizing a Web API service

### Creating a docker file
First we need to specify what we want to have in our container. For this we create a file named `Dockerfile` in our project directory.
```yaml
# Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.2 AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/core/aspnet:2.2
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "identity.dll"]
```
With the `FROM` command we tell Docker that we want to build on an existing image (more about that later). This specific image, more appropriately its tag, is a multi-arch one which means it will be Linux or Windows based depending on the host it is run on. Alternatively we could have specified a specific OS, for example the following will use an Ubuntu 18.04 as its base
```yaml
FROM mcr.microsoft.com/dotnet/core/sdk:2.2-bionic AS build-env
```

Over this base layer, we layer our app. We do this with the line `COPY . ./`. It copies everything from our project directory to the container, to ignore certain files or directories we can add a `.dockerignore` file to our project.
```yaml
# .dockerignore
bin\
obj\
```

### Building a docker image
Before we proceed, we make sure Docker is installed and running.

Our docker file is a template for a Docker-image. With the following command we can create this image. We can use the Visual Studio Code terminal again.
```shell
docker build -t identity
```
This will download the base image, and add our app to a new image.

With `docker images` we can list all images that we've already donwloaded.

### Running a docker image/container
Now that we have successfully created our image we can start a container (an instance of the image).
``` shell
docker run -d -p 8080:80 --name identityinstance identity
```
We tell Docker to forward the host port 8080 to the container port 80.
Next we check if the container is running.
```shell
docker ps
> CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
> 2a304a757a86        identity            "dotnet identity.dll"    19 seconds ago      Up 18 seconds       0.0.0.0:8080->80/tcp   identityinstance
```

So we should be able to access our Web API service through http://localhost:8080/api/values and get back our two values again.
```json
["value1","value2"]
```

To stop our container 
```shell
docker stop identityinstance
```
to start it again
```shell
docker start identityinstance
```

We can also execute apps in our container, for example to list the contents of our working directory we can use
```shell
docker exec identityinstance ls
> Microsoft.AI.DependencyCollector.dll
> Microsoft.ApplicationInsights.AspNetCore.dll
> Microsoft.ApplicationInsights.dll
> ...
```

[Project available on GitHub](https://github.com/00101010/microservices)

# References
* [Tutorial: Create a web API with ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/tutorials/first-web-api?view=aspnetcore-2.2&tabs=visual-studio-code)
* [Dockerize a .NET Core application](https://docs.docker.com/engine/examples/dotnetcore/)
* [[Youtube] Kubernetes for Beginners - Docker Introduction](https://www.youtube.com/watch?v=rmf04ylI2K0)