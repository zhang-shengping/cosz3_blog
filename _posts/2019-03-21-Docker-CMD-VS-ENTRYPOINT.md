---
title: Docker CMD vs ENTRYPOINT
categories:
- Docker
tags:
- docker
- self-note
---

# CMD VS ENTRYPOINT

1. 单独使用 `CMD` 或者 `ENTRYPOINT` 都可以在 Dockerfile 中定义多次，但是每次只有最后定义的那一个生效。

2. 在只使用 `CMD`时候 (最好使用 `exec` 形式，而不是 `shell`)， `CMD` 可以声明一个 executable 也可以声明一个 default arguments。`CMD` 在作为 default arguments 时候需要和 `ENTRYPOINT` 连用。

   > **The main purpose of a CMD is to provide defaults for an executing container.** These defaults can include an executable, or they can omit the executable, in which case you must specify an `ENTRYPOINT` instruction as well.

   注意: 当 `CMD` 做 executable 使用时候，`CMD` 中声明的的 command 可以被`docker run contain_xxx command`  中的 commnad 覆盖

3. 在只使用 `ENTRYPOINT` 时候 (最好使用 `exec` 形式，而不是 `shell`)，docker container 被当做一种可执行的文件来使用 (`ENTRYPOINT`  需要定义一个 executable)。

   注意 `ENTRYPOINT` 不会被（ `docker run contain_xxx command` ）commnad 覆盖，只用使用 `--entrypoint` 的方式覆盖。

   > An `ENTRYPOINT` allows you to configure a container that will run as an executable.
   >
   > You can override the `ENTRYPOINT` instruction using the `docker run --entrypoint` flag

4. 当同时使用  `ENTRYPOINT` 和 `CMD` 时，`ENTRYPOINT` 不可以使用 `shell` 方式会发生错误：

   ```dockerfile
   FROM ubuntu
   ENTRYPOINT top -b
   CMD --ignored-param1
   ```

   ```shell
   $ docker run -it --name test top --ignored-param2
   Mem: 1704184K used, 352484K free, 0K shrd, 0K buff, 140621524238337K cached
   CPU:   9% usr   2% sys   0% nic  88% idle   0% io   0% irq   0% sirq
   Load average: 0.01 0.02 0.05 2/101 7
     PID  PPID USER     STAT   VSZ %VSZ %CPU COMMAND
       1     0 root     S     3168   0%   0% /bin/sh -c top -b cmd cmd2
       7     1 root     R     3164   0%   0% top -b
   ```

   >You can see from the output of `top` that the specified `ENTRYPOINT` is not `PID 1`.

   > If you then run `docker stop test`, the container will not exit cleanly - the `stop` command will be forced to send a `SIGKILL` after the timeout:

   ```sh
   $ docker exec -it test ps aux
   PID   USER     COMMAND
       1 root     /bin/sh -c top -b cmd cmd2
       7 root     top -b
       8 root     ps aux
   $ /usr/bin/time docker stop test
   test
   real	0m 10.19s
   user	0m 0.04s
   sys	0m 0.03s
   ```



   这时必须在 `ENTRYPOINT shell` 声明中加上 `exec` 字段。

   ```dockerfile
   FROM ubuntu
   ENTRYPOINT exec top -b
   ```

   > When you run this image, you’ll see the single `PID 1` process:

   ```shell
   $ docker run -it --rm --name test top
   Mem: 1704520K used, 352148K free, 0K shrd, 0K buff, 140368121167873K cached
   CPU:   5% usr   0% sys   0% nic  94% idle   0% io   0% irq   0% sirq
   Load average: 0.08 0.03 0.05 2/98 6
     PID  PPID USER     STAT   VSZ %VSZ %CPU COMMAND
       1     0 root     R     3164   0%   0% top -b
   ```

   > Which will exit cleanly on `docker stop`:

   ```shell
   $ /usr/bin/time docker stop test
   test
   real	0m 0.20s
   user	0m 0.02s
   sys	0m 0.04s
   ```



5. 当同时使用  `ENTRYPOINT` 和 `CMD` 时，一般使用规则是 `CMD` 充当 `ENTRYPOINT` 的默认参数。注意种情况下 `ENTRYPOINT` 和 `CMD` 必须使用 `exec` 格式。

   |                                | No ENTRYPOINT              | ENTRYPOINT exec_entry p1_entry | ENTRYPOINT [“exec_entry”, “p1_entry”]          |
   | :----------------------------- | :------------------------- | :----------------------------- | :--------------------------------------------- |
   | **No CMD**                     | *error, not allowed*       | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry                            |
   | **CMD [“exec_cmd”, “p1_cmd”]** | exec_cmd p1_cmd            | /bin/sh -c exec_entry p1_entry | ***exec_entry p1_entry exec_cmd p1_cmd***      |
   | **CMD [“p1_cmd”, “p2_cmd”]**   | p1_cmd p2_cmd              | /bin/sh -c exec_entry p1_entry | ***exec_entry p1_entry p1_cmd p2_cmd***        |
   | **CMD exec_cmd p1_cmd**        | /bin/sh -c exec_cmd p1_cmd | /bin/sh -c exec_entry p1_entry | exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd |

6. Practical way:

   ```dockerfile
   FROM ubuntu
   ENTRYPOINT ["top", "-b"]
   CMD ["-c"]
   ```



References:

官方：https://docs.docker.com/engine/reference/builder/#entrypoint

Blog：https://medium.freecodecamp.org/docker-entrypoint-cmd-dockerfile-best-practices-abc591c30e21

​	   http://goinbigdata.com/docker-run-vs-cmd-vs-entrypoint/


