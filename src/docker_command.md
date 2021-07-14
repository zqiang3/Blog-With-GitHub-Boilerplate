---
layout: post
title: docker
slug: docker
date: 2021-07-14 19:57:36
status: publish
author: 君祁
categories:
  - 容器技术
tags:
  - docker
excerpt: docker command
---

## docker commands
* attach: Attach local standard input, output, and error streams to a running container
* build: Build an image from a Dockerfile
* commit: Create a new image from a container's changes
* create: Create a new container
* diff: Inspect changes to files or directories on a container's filesystem
* exec: Run a command in a running container
* images: List images
* info: Display system-wide information
* inspect: Return low-level information on Docker objects
* kill: Kill one or more running containers
* login: Log in to a Docker registry
* logout: Log out from a Docker registry
* logs: Fetch the logs of a container
* pause: Pause all processes within one or more containers
* ps: List containers
* pull: Pull an image or a repository from a registry
* push: Push an image or a repository to a registry
* rename: Rename a container
* restart: Restart one or more containers
* rm: Remove one or more containers
* rmi: Remove one or more images
* run: Run a command in a new container
* search: Search the Docker Hub for images
* start: Start one or more stopped containers
* stats: Display a live stream of container(s) resource usage statistics
* stop: Stop one or more running containers
* tag: Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
* top: Display the running processes of a container
* unpause: Unpause all processes within one or more containers
* update: Update configuration of one or more containers
* version: Show the Docker version information
* wait: Block until one or more containers stop, then print their exit codes

## docker run
使用docker run的时候，通过-v HOST_DIR:CONTAINER_DIR参数来指定数据卷使用的主机目录，这个做法一般被称为“绑定挂载”。
主机目录指运行docker引擎的机器。如果是远程连接到docker服务的话，那么远端的计算机必须已经存在该路径。