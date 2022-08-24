# Debugging Docker

## Learning Goals

- Understanding common debugging methods for Docker

## Instructions

While containerizing applications does tend to lead to better reproducibility and cross-platform functionality,
it does also introduce its own host of problems. If your application does not work as expected after converting to a container,
you now have the additional bug surface area of an entire Linux OS, a virtualized networking stack and environment, Dockerfiles and
Docker Compose files, and also the added complexity of the Docker tool itself.
Debugging problems encountered at this point isn't for the faint of heart, and has a lot of difficult areas to navigate if you are not
familiar with Linux and networking.

We'll be simplifying this process down to a more manageable set of steps to attempt, but ultimately a bug from any of the
previously mentioned areas could end up being encountered.

## Common Debugging Practices with Docker and Containers

### Logging

The primary tool you should almost always be relying on for debugging purposes is the logging output from your application. By default, the Spring
Boot server will be logging its output to the Linux Standard Output(stdout) stream, which can be accessed via Docker's log output.

``` text
docker run --name spring-boot-lab --network labnetwork -d -p 8080:8080 -e SPRING_DATASOURCE_URL=jdbc:postgresql://fake-server:5432/db_test rest-service-complete:0.0.1-SNAPSHOT 

# View logs from crashed container
docker logs spring-boot-lab
```
``` shell
...
org.postgresql.util.PSQLException: The connection attempt failed.
...
Caused by: java.net.UnknownHostException: fake-server
```
``` text
docker rm -f spring-boot-lab
```

This output should be the most informative as to what is happening in your application. And assuming you are responsible for developing the application itself,
extra debug logging can be easily added and enabled as needed.

### Quick Docker Sanity Check

As we progress down this list, each step becomes more and more time consuming to perform and investigate. If nothing helpful is in the logging output, and there are no ways
to gather more information at that level, this is probably a good time to perform some quick sanity checks with the Docker environment itself. We can run some trivial containers to make
sure nothing is interfering with the full environment, or the various components such as the virtual networking. Rebooting a machine at this point probably isn't a bad call either.

``` text
docker run hello-world
```
``` shell
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

``` text
docker run -d -p 8080:80 nginx
curl http://127.0.0.1:8080
```
``` shell
...
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

If any of these quick tests fail, then there should be no need to continue debugging at the application or service level, until the Docker Daemon is back up and running.

### Debugging of Live Running Applications

If the application is misbehaving, but the container is still running, we can open a terminal connection into the container and debug as if it were a live Linux system.

This may not always be easy to do however. Container images tend to be lightweight with few tools installed, but it is normally possible to install extra packages live or build into a debugging
image. Also, application failure modes that lead to this behavior tend to be less common than outright crashes.

``` text
# Exec into previous lab's application
docker exec -u root -it java-mod-10-orchestrating-using-compose_spring-boot-compose_1 /bin/bash
root@5b1116a31966:/# ps auxf
```
``` shell
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         207  0.1  0.0  18516  3368 pts/1    Ss   21:47   0:00 /bin/bash
root         221  0.0  0.0  34416  2844 pts/1    R+   21:47   0:00  \_ ps auxf
cnb            1  0.3  2.3 3047524 372864 ?      Ssl  19:13   0:29 java org.springframework.boot.loader.JarLauncher
```
``` text
# Tools missing
root@5b1116a31966:/# ping postgres-compose-master-1
```
``` shell
bash: ping: command not found
```
``` text
root@5b1116a31966:/# ip addr
```
``` shell
bash: ip: command not found
```
``` text
root@5b1116a31966:/# netstat -aont
```
``` shell
bash: netstat: command not found
```
``` text
# Install tools
root@5b1116a31966:/# apt update
root@5b1116a31966:/# apt install iputils-ping net-tools iproute2
...
# Tools available
root@5b1116a31966:/# ping -c1 postgres-compose-master-1
```
``` shell
PING postgres-compose-master-1 (172.20.0.2) 56(84) bytes of data.
64 bytes from postgres-compose-master-1.java-mod-10-orchestrating-using-compose_default (172.20.0.2): icmp_seq=1 ttl=64 time=0.206 ms

--- postgres-compose-master-1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.185/0.185/0.185/0.000 ms
```
``` text
root@5b1116a31966:/# ip addr
```
``` shell
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
68: eth0@if69: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:14:00:05 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.0.5/16 brd 172.20.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
``` text
root@5b1116a31966:/# netstat -aont
```
``` shell
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       Timer
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 127.0.0.11:32905        0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 172.20.0.5:42586        172.20.0.2:5432         ESTABLISHED off (0.00/0/0)
```

All this would be needed just to get a system into a state where further debugging can be done. From here would be Linux systems, and application framework specific debugging.

### Launching Docker Containers with Modified Entrypoints

If the application container is immediately crashing, and has no logging output, none of the above steps will work. In these cases, a common practice is to run a new container,
but override the Entrypoint with a shell command. Once the container starts, the application itself can be launched manually from inside the container, and observed for any
problems.
The biggest negative with this approach tends to be with third party applications, as it can take some work to determine how the application is being executed.

``` text
docker run --name spring-boot-lab --network labnetwork -p 8080:8080 -e SPRING_DATASOURCE_URL=jdbc:postgresql://fake-server:5432/db_test --entrypoint /bin/bash -it rest-service-complete:0.0.1-SNAPSHOT
cnb@dee77395a2dd:/$ ps auxf
```
``` shell
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
cnb            1  0.3  0.0  18520  3456 pts/0    Ss   15:40   0:00 /bin/bash
cnb           11  0.0  0.0  34416  2752 pts/0    R+   15:40   0:00 ps auxf
```
``` text
cnb@dee77395a2dd:/$ cd /workspace/
cnb@dee77395a2dd:/workspace$ ls
```
``` shell
BOOT-INF  META-INF  org
```
``` text
cnb@dee77395a2dd:/workspace$ JAVA_HOME=/layers/paketo-buildpacks_bellsoft-liberica/jre PATH=/layers/paketo-buildpacks_bellsoft-liberica/jre/bin:$PATH java org.springframework.boot.loader.JarLauncher
```
``` shell

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.6.3)


...

```

### Next Steps


If none of the previous steps resolve the issue at hand, then there really are no well defined processes to follow for further debugging. At this point, the problem could be at any point of the full
technology stack, and may require debugging through all those elements.

Some common problems to keep in mind though:

- Disk space filled
  
In use systems tend to fill up on disk space when not being perfectly managed. This frequently is a hard breaker for Docker systems, so is normally a good item to check on.
  
- Permissions conflicts with volumes

Docker works by isolating an entire OS. Including permissions. There are commonly issues where both the containerized OS and the host OS are having permission conflicts on the same underlying files.
If you are using volume or bind mounts, this will be an item to keep an eye on.

- Unexpected changes to running images

Running images can be updated to breaking version either on purpose, or accidentally. Any change, no matter how apparently insignificant, can break a system. Try to roll back and test previous know
working versions if possible, and try to use explicit tagging where possible to prevent accidental image updates.

- Buggy Docker daemon/software/kernel features

All software has bugs. Including the Docker Daemon, and Linux kernel features it relies upon.
If observed Docker behavior seems to be completely unexplainable, it might be worth while upgrading/downgrading Docker systems, and searching online for any related bug reports.
