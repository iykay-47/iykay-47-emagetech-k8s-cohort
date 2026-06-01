## Assignment 01 — CLI Essentials & Docker Commands
## Part 1:
### Which CLI command was new to you, and what did you use it for?

Using head and tail. I usually pipe an output into the commands as input e.g 

``cat greeting.txt | head -10``

Inputing a file directly is shorter and more neat. i.e

``head -1 greeting.txt``  

## Part 2

### Q1: What is the difference between rm file.txt and rm -rf directory/? Why is the second form considered dangerous?

`rm file.txt` removes a single file while `rm -rf directory/` removes a directory and all files within it as long as user has write permissions on said directory.
`rm -rf directory/` is dangerous as Linux system does not have a trash feature and deletes file entirely from the system disk.

### Q2: After running the four commands above, how many images do you have? How many containers? Why?
```
➜  ~ docker image pull alpine:3.19
3.19: Pulling from library/alpine
17a39c0ba978: Pull complete 
Digest: sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
Status: Downloaded newer image for alpine:3.19
docker.io/library/alpine:3.19
```

```
➜  ~ docker image ls

IMAGE         ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine:3.19   83b2b6703a62        7.4MB             0B        
```

```
➜  ~ docker container run alpine echo hi
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
6a0ac1617861: Pull complete 
Digest: sha256:5b10f432ef3da1b8d4c7eb6c487f2f5a8f096bc91145e68878dd4a5019afde11
Status: Downloaded newer image for alpine:latest
hi
```

```
➜  ~ docker container ls -a
CONTAINER ID   IMAGE     COMMAND     CREATED              STATUS                          PORTS     NAMES
37c7a5e47985   alpine    "echo hi"   About a minute ago   Exited (0) About a minute ago             focused_kapitsa
```

```
➜  ~ docker images  
IMAGE           ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine:3.19     83b2b6703a62        7.4MB             0B        
alpine:latest   3cb067eab609       8.45MB             0B    U   
```
#### 2 Images and 1 Container.
The docker pull command is run with a with a specific version of alpine, hence docker pulls that specific image. However when when `docker container run` is done with alpine and so specific image tag, docker searches the local disk first for an alpine image with tag `alpine:latest`, since alpine 3.19 no longer has the tag latest, docker pulls a new image from dockerhub with the tag latest to create a container. This is why specific tag should be used when pulling and psuhing image to avoid playing roullete with in Prod.

1 container in exited status.This is because since there is no long running process, the container exits. The container still shows up in exited status because docker persist the image date of the container until explictly removed with the `rm` command.


### Q3: What's the difference between docker run -it alpine sh and docker exec -it <name> sh? When would you use each?

`docker run -it alpine sh` will create a new container from the alpine image then stay on the interactive shell after the container is created `docker exec -it <name> sh` needs an already running container to execute on. It will return an error if container is in an exited status or doesn't exist.


## Part 3
```
➜  ~ docker container ls
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                                     NAMES
c126755ebde4   nginx:1.25   "/docker-entrypoint.…"   16 seconds ago   Up 15 seconds   0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   practice-web
```

**Container name = practice web**

**Host port 8081 binded to container port 80**

**Image used = nginx:1.25**

After running echo "Heloo from $STUDENT_NAME" > /usr/share/nginx/html/index.html
```
~ curl localhost:8081
Heloo from Ike
```
Ip-Address of the container after mapping the JSON output of `docker container inspect practice-web`

```
~ docker container inspect -f {{.NetworkSettings.Networks.bridge.IPAddress}} practice-web
172.17.0.2
```


## DOCKER SURPRISE
The one good suprise about docker is the docker command structute.

It uses most of linux basic command for options such as ls, rm , ps, logs etc which makes it easier to form a mental model of how to create and remember commands especially for debugging and monitoring.