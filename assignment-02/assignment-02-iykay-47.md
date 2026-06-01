# Assignment 02 — Ike

**GitHub username:** <your-username>
**Date completed:** YYYY-MM-DD
**Language chosen:** Python | Node.js

## 1. The image I built

- Final image ID: `sha256:a0e22a1d85af42e65dbdb1b4c5908268ab14d84780dadf9bb407a01245322f75`
- Image size: `148MB`
- Number of layers (`docker image history cohort-greet:0.2.0 | tail +2 | nl | wc -l`): `16`

### Dockerfile
```
# syntax=docker/dockerfile:1

# Set and activate buildkit
# Set base Image

FROM python:3.11-slim
RUN apt update && apt-get install -y --no-install-recommends procps
WORKDIR /app 
COPY app.py .

ENV PORT=8000
EXPOSE 8000

CMD [ "python", "app.py" ]

```

### .dockerignore

```.git
.gitignore
__pycache__
*.pyc
*.log
README.md
assignment-02-iykay-47.md
```

## 2. Answers to the 8 questions

**Q1 — what `.dockerignore` affects?:**
.dockerigonre affects every file/file path listed within it. It prevents such files from been loaded to the docker binary during the build process hence increasing the size of the image even if it has no use or implementation within the Dockerfile.

As such every single file not invovled in the dockerbuild process should be listed in it to reduce image size.

**Q2 — what is the image ID a hash of:** 

```
➜  assignment-02 docker image ls cohort-greet
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
cohort-greet:0.1.0   ac27a52e1b3d        124MB             0B    
```
The Image ID is actually a hash of the local copy of the Dockerfile(manifest) implementation(layer set).
Hence any change within the Dockerfile/manifest will lead to a change in the layer set as well as the hash/ID and a new image.

**Q3 — largest layer and why:**

The layer is the FROM instruction, which involved downloading the entire baseimage from the docker registry as it was not found locally.

**Q4 — `--memory 64m` shows up as what value:**

`67108864` It shows up diffrenet as {{.HostConfig.Memory}} output from docker inspect is in bytes and not MiB or more human readable.
```
➜  assignment-02 docker container inspect -f '{{.HostConfig.Memory}}' greet
67108864
```
**Q5 — PID of my app inside the container:**

PID: 1
Reason might be because it is the first long running process from when image was created.

```
➜  assignment-02 docker container exec -it greet sh
# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.2  0.0  26468 17536 ?        Ss   18:21   0:00 python app.py
```


**Q6 — `stop` vs `kill`, and which for a database:**

Stop tries to gracefully shuts down processes running within the container(SIGTERM) before stopping the container reducing potential of data loss while kill forcefully shuts down the container regardless of any process running within it.

Stop is best used with databases to prevent massive data loss and potential data corruption.
**Q7 — what same-IMAGE-ID-across-tags proves:**

```
➜  assignment-02 docker image ls
IMAGE                 ID             DISK USAGE   CONTENT SIZE   EXTRA
cohort-greet:0.1      ac27a52e1b3d        124MB             0B        
cohort-greet:0.1.0    ac27a52e1b3d        124MB             0B        
cohort-greet:0.2.0    a0e22a1d85af        148MB             0B    U   
cohort-greet:latest   ac27a52e1b3d        124MB             0B  
```

This proves ehile docker tags may change for an image, they are still tied to the digest/Id of an original image hence should be used carefully and appropriately.

Tags are muteable, digests are not.

**Q8 — tag vs digest mutability: If Docker Hub re-tagged alpine:3.19 tomorrow to point at a different layer set, would alpine:3.19 and alpine@sha256:... (the digest) give you the same image? Explain in two sentences.**
```
➜  assignment-02 docker image ls --digests alpine     
REPOSITORY   TAG       DIGEST                                                                    IMAGE ID       CREATED        SIZE
alpine       3.19      sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1   83b2b6703a62   7 months ago   7.4MB
```

No. This is because the a new layer set will lead to creation of a new hash identifier for a new image.
As such while the tags looks unchanged, the digest of the new layer set will be different hence pointing at different images.  

## 3. Evidence

- `docker image history cohort-greet:0.1.0`
```
➜  assignment-02 docker image history cohort-greet:0.1.0 
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
ac27a52e1b3d   17 minutes ago   CMD ["python" "app.py"]                         0B        buildkit.dockerfile.v0
<missing>      17 minutes ago   EXPOSE [8000/tcp]                               0B        buildkit.dockerfile.v0
<missing>      17 minutes ago   ENV PORT=8000                                   0B        buildkit.dockerfile.v0
<missing>      17 minutes ago   COPY app.py . # buildkit                        740B      buildkit.dockerfile.v0
<missing>      35 hours ago     WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      8 days ago       CMD ["python3"]                                 0B        buildkit.dockerfile.v0
<missing>      8 days ago       RUN /bin/sh -c set -eux;  for src in idle3 p…   36B       buildkit.dockerfile.v0
<missing>      8 days ago       RUN /bin/sh -c set -eux;   savedAptMark="$(a…   42MB      buildkit.dockerfile.v0
<missing>      8 days ago       ENV PYTHON_SHA256=272179ddd9a2e41a0fc8e42e33…   0B        buildkit.dockerfile.v0
<missing>      8 days ago       ENV PYTHON_VERSION=3.11.15                      0B        buildkit.dockerfile.v0
<missing>      8 days ago       ENV GPG_KEY=A035C8C19219BA821ECEA86B64E628F8…   0B        buildkit.dockerfile.v0
<missing>      8 days ago       RUN /bin/sh -c set -eux;  apt-get update;  a…   3.81MB    buildkit.dockerfile.v0
<missing>      8 days ago       ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
<missing>      8 days ago       ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
<missing>      10 days ago      # debian.sh --arch 'amd64' out/ 'trixie' '@1…   78.6MB    debuerreotype 0.17
```
- `docker container run` (Part 2.2 — the detached run with all flags)

```
➜  assignment-02 docker container run -d \
--name greet \
-p 8080:8000 \
-e STUDENT_NAME="Ike" \
-e GREETING="Hi" \
--restart unless-stopped \ 
--memory 64m \ 
--cpus 0.25 \
> cohort-greet:0.1.0 
3ec117ce7321f692670dfa0324a9c2e4fb3555816b10348c3bb66efe318d96d5
```
- `docker container logs greet` after 2 curl requests

```
➜  ~ docker container logs greet
listening on :8000
[req] 172.17.0.1 "GET / HTTP/1.1" 200 -
[req] 172.17.0.1 "GET / HTTP/1.1" 200 -
```
- `docker container stats --no-stream greet`

```
➜  ~ docker container stats --no-stream greet 
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O           BLOCK I/O     PIDS
df0011c930be   greet     0.01%     13.4MiB / 64MiB     20.94%    8.08kB / 1.29kB   0B / 1.95MB   1

```
- `docker container inspect -f '{{.HostConfig.RestartPolicy.Name}} {{.HostConfig.Memory}}' greet`

```
➜  ~ docker container inspect -f '{{.HostConfig.RestartPolicy.Name}} {{.HostConfig.Memory}}' greet
unless-stopped 67108864
```
- `docker image ls cohort-greet` (showing the three tags from Part 3.1)
- (Optional) URL of your pushed image on Docker Hub / GHCR

## 4. One thing that surprised me

(2–4 sentences. Be specific — "the image was bigger than I expected because the Python base image alone is 130MB and my app is ~2KB" is a good answer; "Docker is cool" is not.)
The memory limit kept me from installing procps tool in the container. The process kept getting Killed(OOM) before installation process completion.

```
# apt install -y --no-install-recommends procps
Installing:                     
  procps
...

debconf: unable to initialize frontend: Dialog
debconf: (No usable dialog-like program is installed, so the dialog based frontend cannot be used. at /usr/share/perl5/Debconf/FrontEnd/Dialog.pm line 79, <STDIN> line 2.)
debconf: falling back to frontend: Readline
debconf: unable to initialize frontend: Readline
debconf: (Can't locate Term/ReadLine.pm in @INC (you may need to install the Term::ReadLine module) (@INC entries checked: /etc/perl /usr/local/lib/x86_64-linux-gnu/perl/5.40.1 /usr/local/share/perl/5.40.1 /usr/lib/x86_64-linux-gnu/perl5/5.40 /usr/share/perl5 /usr/lib/x86_64-linux-gnu/perl-base /usr/lib/x86_64-linux-gnu/perl/5.40 /usr/share/perl/5.40 /usr/local/lib/site_perl) at /usr/share/perl5/Debconf/FrontEnd/Readline.pm line 8, <STDIN> line 2.)
debconf: falling back to frontend: Teletype
Killed
```

Had to rebuild a new image with new Dockerfile to prevent increasing memory size allocation.


Adding 23.5MB to to image size
```
➜  ~ docker image history cohort-greet:0.2.0 | nl   
     1	IMAGE          CREATED             CREATED BY                                      SIZE      COMMENT
     2	a0e22a1d85af   About an hour ago   CMD ["python" "app.py"]                         0B        buildkit.dockerfile.v0
     3	<missing>      About an hour ago   EXPOSE [8000/tcp]                               0B        buildkit.dockerfile.v0
     4	<missing>      About an hour ago   ENV PORT=8000                                   0B        buildkit.dockerfile.v0
     5	<missing>      About an hour ago   COPY app.py . # buildkit                        740B      buildkit.dockerfile.v0
     6	<missing>      About an hour ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
     7	<missing>      About an hour ago   RUN /bin/sh -c apt update && apt-get install…   23.5MB    buildkit.dockerfile.v0
     8	<missing>      8 days ago          CMD ["python3"]                                 0B        buildkit.dockerfile.v0
     9	<missing>      8 days ago          RUN /bin/sh -c set -eux;  for src in idle3 p…   36B       buildkit.dockerfile.v0
    10	<missing>      8 days ago          RUN /bin/sh -c set -eux;   savedAptMark="$(a…   42MB      buildkit.dockerfile.v0
    11	<missing>      8 days ago          ENV PYTHON_SHA256=272179ddd9a2e41a0fc8e42e33…   0B        buildkit.dockerfile.v0
    12	<missing>      8 days ago          ENV PYTHON_VERSION=3.11.15                      0B        buildkit.dockerfile.v0
    13	<missing>      8 days ago          ENV GPG_KEY=A035C8C19219BA821ECEA86B64E628F8…   0B        buildkit.dockerfile.v0
    14	<missing>      8 days ago          RUN /bin/sh -c set -eux;  apt-get update;  a…   3.81MB    buildkit.dockerfile.v0
    15	<missing>      8 days ago          ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
    16	<missing>      8 days ago          ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
    17	<missing>      10 days ago         # debian.sh --arch 'amd64' out/ 'trixie' '@1…   78.6MB    debuerreotype 0.17


```

## 5. One thing I'm still unsure about

Does each tag create a copy of a docker image or a link to the docker image digest/Id.
Since docker builds with instruction from the Dockerfile, how does the image file size end up been affected by extra filesin the directory not explicitly mentioned in the  .dockerignore file.