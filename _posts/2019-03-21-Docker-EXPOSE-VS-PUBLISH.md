---
title: Docker EXPOSE vs PUBLISH
categories:
- Docker
tags:
- docker
- self-note
---

# EXPOSE vs PUBLISH

1. 检查一个 container 的 port mapping 使用命令 `docker port CONTAINER[PRIVATE_PORT[/PROTOCOL]]`

   > ```shell
   > docker port test_container
   > 7890/tcp -> 0.0.0.0:4321
   > 9876/tcp -> 0.0.0.0:1234
   > 
   > docker port test_container 7890/tcp
   > 0.0.0.0:4321
   > ```

2. 在 Dockerfile 中使用 `EXPOSE` 声明一个 `PRIVATE_PORT`，这个 Port 不能和其他网络或者宿主机网络建立连接。

   ```dockerfile
   EXPOSE <port> [<port>/<protocol>...]
   ```

   > But **EXPOSE will not** allow communication via the defined ports to containers outside of the same [network](https://docs.docker.com/network/) or to the host machine. To allow this to happen you need to *publish* the ports.

3. 也可以使用 `--expose` 方式声明 `PRIVATE_PORT`  , `docker run --expose=1234 my_app`

4. `PUBLISH` 用来发布 container   `PRIVATE_PORT` 到宿主机上，这样可以和宿主机建立网络连接，或者访问外部网络。 `PUBLISH` 有两个参数 `-P` 和 `-p`

   `-P`: 发布所有的 exposed `PRIVATE_PORT` 如：`docker run -P my_app`

   `-p`: 发布某个 exposed `PRIVATE_PORT` 如： `docker run -p 80:80/tcp -p 80:80/udp my_app`

