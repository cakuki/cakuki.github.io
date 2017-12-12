---
layout: single
title: Docker for Efficent Node.js Development
date: 2017-12-10 13:35:55 +0200
categories: tech
---


I assume you have already heard of Docker or are using it everyday. There are lots of articles and blog posts about advantages and techniques for making your images smaller, faster, stronger.

But for daily Node.js development cycle, Docker could be too long to build or your images are too distinct from your production version.

This post covers some basic tricks to have faster code & test cycle.

---

## Prerequisite: Folder Structure

For making these tricks to work, your Node.js code should reside under a specific directory like src. Which is also helpful for distinguishing your original code from dependencies (node_modules), tests and metafiles like package.json, .eslint and so on. I personally believe this is a good convention for all Node.js (or even all JavaScript) projects.

## Use layering

For speeding up Docker builds, the best way is using Docker’s layer caching. In short, for each directive in Dockerfile, Docker creates another layer over the base layer (or base image). And given same files over same layer, Docker is smart enough to use a layer that was build and cached before. So if you build this Dockerfile twice you would see that Docker builds it from cache for the second time:

```
# Dockerfile
FROM node:8
COPY package.json .
RUN npm install
```

And although npm install footprint may change (new versions might be fetched from unfixed  dependencies), Docker will use same cached layer for npm install command as long as the package.json file and RUN command stay same.

```Dockerfile
# Dockerfile
FROM node:8
WORKDIR /app
COPY package.json .
RUN npm install
COPY src .
```

So in order to use caching/layering well, commands in Dockerfile should be ordered in a way that most frequently changed parts are included in later statements. So I always try to keep COPY src . as my last statement.

## Use mounting

Another way to speed things up in development, is to have another Dockerfile just for development. You might want to keep development & production Dockerfiles same for integrity, but as long as the build process is the same, mounting source code will not effect runtime behaviour.

```Dockerfile
# Dockerfile.development
FROM node:8
WORKDIR /app
VOLUME /app/src
COPY package.json .
RUN npm install
CMD ["npm", "start"]
```

Then to run from terminal:

```
docker build -t my-node-app -f Dockerfile.development .
docker run -d -v $PWD/src:/app/src --name node-app my-node-app
# make some changes
docker restart node-app
```

## Use auto restart

Unlike the last topic “use mounting” this is a direct change. But if you need to make changes and test it in numerous times, it’s inevitable. I am using nodemon for this purpose as it’s a really simple and straightforward module, there are many alternative for this like pm2, forever, supervisor.

```Dockerfile
# Dockerfile.development
FROM node:8
WORKDIR /app
RUN npm install --global nodemon
COPY package.json .
RUN npm install
CMD ["npm", "start"]
```

So this time, even before adding our package.json we install nodemon and use nodemon instead of node for launching our app.

## Use dockerignore

While developing Node.js apps, it’s inevitable for the package.json grow in size with dependencies. And not very late, node_modules directory grows up in size with dependencies, and their dependencies and theirs.

Some of node modules have native Node.js addons generated (by node-gyp) when they are installed. This step produces binaries that can only be used by the system that they were built on. They cannot be used inside Docker images as they are based from a Linux image. (In a rare case, you could be developing on a Linux system that is identical.) That means, the node_modules directory we have generated in our system is probably useless to Docker image we are building. That’s why all Dockerfile examples above, run npm install.

Every docker build . command transfers all files into Docker Engine, unless they are ignored in a .dockerignore file. And to gain more speed, they should be ignored like:

```
# .dockerignore
.git
node_modules
```

Excluding node_modules directory could save you up lots of time in total. Waste less on “sending context to docker daemon” stage.

## Conclusion

As mentioned above, we don’t need to generate node_modules directory in our environment, which means we don’t even need npm & node binaries installed natively in our system.

That brings freedom to distribute your work with less technical people on your team.

Dockerized development deals with managing node versions (via nvm, n, etc.) for you and has same speed as native with these tricks. And with the power of Docker Composer, you could create other linked services (like mongo, redis…) with ease and forget about installing server software and managing their versions manually.
