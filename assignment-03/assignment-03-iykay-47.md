# Assignment 03 — Ike

**GitHub username: iykay-47**

**Date completed: 2026-05-31**

**Git SHA of submitted app: e077d88**

## 1. Size comparison table

| Variant            | Size  | Layers | Stop time | Exit code |
|--------------------|-------|--------|-----------|-----------|
| `cohort-greet:naive` | 155 MB | 16     | 5.41 s     | 137         |
| `cohort-greet:multi` | 741 MB | 21     | 0.26 s     | 0         |

(Layers = output of `docker image history <tag> | wc -l` minus 1 for the header.)

## 2. Final image digest

`sha256:f7babed1753a72fff28ac221ddaa6b783f037e04ff8493bfee5db8904e2cf794`

## 3. Answers to the 7 questions

**Q1 — naive size + stop behaviour + why:**

Image size and exit code shown below.

Container Stop time : 5.145s 
```
➜  ~ docker image ls cohort-greet:naive --format 'naive | {{.Size}}' 
naive | 741MB

...

➜  ~ time docker container stop --timeout 5 greet-naive                 
greet-naive
docker container stop --timeout 5 greet-naive  0.01s user 0.01s system 0% cpu 5.145 total


➜  ~ docker container inspect greet-naive --format 'ExitCode={{.State.ExitCode}}' 
ExitCode=137
```
Since the image build used shell in executing the gunicorn process, the parent process isn't gunicorn but shell and may not send the SIGTERM call to it's child process. Hence after timeout is exhausted, docker sends a SIGKILL command which bypasses gaceful shutdown of any current processes.

**Q2 — build output, CACHED vs rebuilt:** ...

```
➜  assignment-03 git:(main) ✗ docker image build -t cohort-greet:multi .
[+] Building 1.3s (16/16) FINISHED
...

 => CACHED docker-image://docker.io/docker/dockerfile:1.7@sha256:a57df69d0ea827fb7266491f2813635de6f17269be881f696fbfdf2d83dda33e                          0.0s
 => [internal] load metadata for docker.io/library/python:3.11-slim                                                                                        0.4s
 => [internal] load .dockerignore                                                                                                                          0.0s
 => => transferring context: 102B                                                                                                                          0.0s
 => [build 1/5] FROM docker.io/library/python:3.11-slim@sha256:a3ab0b966bc4e91546a033e22093cb840908979487a9fc0e6e38295747e49ac0                            0.0s
 => [internal] load build context                                                                                                                          0.0s
 => => transferring context: 478B                                                                                                                          0.0s
 => CACHED [build 2/5] RUN python -m venv /opt/venv                                                                                                        0.0s
 => CACHED [build 3/5] WORKDIR /app                                                                                                                        0.0s
 => CACHED [build 4/5] COPY requirements.txt .                                                                                                             0.0s
 => CACHED [build 5/5] RUN pip install --no-cache-dir -r requirements.txt                                                                                  0.0s
 => CACHED [runtime 2/5] COPY --from=build /opt/venv /opt/venv                                                                                             0.0s
 => CACHED [runtime 3/5] RUN <<EOF (groupadd appmgt...)                                                                                                    0.0s
 => CACHED [runtime 4/5] WORKDIR /app                                                                                                                      0.0s
 => [runtime 5/5] COPY app.py .  
```

All layers from the previous build were cached except the copy layer which was rebuilt due to a change in the app.py file.
Also no change from the build stage as it doesn't touch the python runtime code.


**Q3 — new stop time/exit + which change:**

```
➜  assignment-03 git:(main) ✗ time docker container stop --timeout 5 greet-multi 
greet-multi
docker container stop --timeout 5 greet-multi  0.02s user 0.01s system 11% cpu 0.259 total
➜  assignment-03 git:(main) ✗ docker container inspect greet-multi --format 'ExitCode={{.State.ExitCode}}'
ExitCode=0
```
New Stop time = 0.26s
Change ocurred from running the CMD in exec form rather than shell form. As such SIGTERM was able to access the process as it will have PID=1  

**Q4 — size reduction breakdown:**

```
➜  assignment-03 git:(main) ✗ docker image ls --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}' | grep cohort-greet
cohort-greet   multi     155MB
cohort-greet   naive     741MB
```

Image shrunk by more than 70%.
Two reasons why this happened:
- Switichng from Ubuntu OS image to a dedicated python-slim image reduced the amount of layers required to install python as well as other configs for python.
- Separating the build stage from the runtime stage and copying only the artifact reqired at runtime from the build stage cuts out atleast 3 layers from the image build process wich in turn reduces the size of the image.
- Minimum amount of RUN layes in the new Dockerfile. 

**Q5 — cache-mount timings + CI relevance:**

```
➜  assignment-03 git:(main) ✗ time docker image build --no-cache -t cohort-greet:multi . 2>&1 
[+] Building 32.7s (16/16) FINISHED
docker image build --no-cache -t cohort-greet:multi . 2>&1  1.48s user 0.59s system 6% cpu 33.820 total

➜  assignment-03 git:(main) ✗ time docker image build --no-cache -t cohort-greet:multi . 2>&1
[+] Building 29.3s (16/16) FINISHED
...
docker image build --no-cache -t cohort-greet:multi . 2>&1  1.37s user 0.56s system 6% cpu 30.384 total

```
A total of about 2s was saved. Which means a warm cache saves more time during the CI process.

**Q6 — secret marker + what `ARG` would leak:**

```
➜  assignment-03 git:(main) ✗ docker container run --rm --name test cohort-greet:secret cat /where-token-was-used                   
Token starts with a80d

➜  assignment-03 git:(main) ✗ docker image history --no-trunc cohort-greet:secret | grep -i "PYPI_TOKEN" && echo "LEAKED" || echo "no leak"
sha256:65f125cddd048312da3d30a7fa045a89f0ecbfc95d7772eb498e1e4cd7463697   41 minutes ago   RUN /bin/sh -c TOKEN=$(cat /run/secrets/pypi_token) &&     echo "Token starts with ${TOKEN:0:4}" > /where-token-was-used # buildkit   23B       buildkit.dockerfile.v0
LEAKED

➜  assignment-03 git:(main) ✗ docker image history --no-trunc cohort-greet:secret | grep -i "arg" && echo "LEAKED" || echo "no leak" 
<missing>                                                                 6 minutes ago   ARG TOKS                                                                                                                                           0B        buildkit.dockerfile.v0
LEAKED

➜  assignment-03 git:(main) ✗ docker image history --no-trunc cohort-greet:secret | grep -i "toks" && echo "LEAKED" || echo "no leak"
sha256:ac521b6b10405b789d63dbf6ec1eb79e18c871e372075abd37a99661407f9596   6 minutes ago   RUN |1 TOKS=test /bin/sh -c echo "token is ${TOKS}" > /test-out # buildkit                                                                         14B       buildkit.dockerfile.v0
<missing>                                                                 6 minutes ago   RUN |1 TOKS=test /bin/sh -c TOKEN=$(cat /run/secrets/pypi_token) &&     echo "Token starts with ${TOKEN:0:4}" > /where-token-was-used # buildkit   23B       buildkit.dockerfile.v0
<missing>                                                                 6 minutes ago   ARG TOKS                                                                                                                                           0B        buildkit.dockerfile.v0
LEAKED
```

While both showed leaked, the value/content of pypi was not leaked into docker history. However that of TOKS was leaked and can be easily seen. Hence a simple search for ARGS can list the name of various secrets to search for is it is used in passing secrets into the docker build/runtime even if the value is passed from the command line during build.

**Q7 — tag vs digest for k8s manifest:** ...

```
➜  ~ docker image ls cohort-greet --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}'
REPOSITORY     TAG             IMAGE ID
cohort-greet   secret          ac521b6b1040
cohort-greet   0.1.0           f7babed1753a
cohort-greet   0.1.0-e077d88   f7babed1753a
cohort-greet   git-e077d88     f7babed1753a
cohort-greet   multi           f7babed1753a
cohort-greet   naive           82c77d432474

➜  assignment-03 git:(main) ✗ docker image inspect cohort-greet:${VERSION} -f '{{.Id}}'
sha256:f7babed1753a72fff28ac221ddaa6b783f037e04ff8493bfee5db8904e2cf794
```

The digest pin is the best option for direct reproducibility to be used in a kubernetes manifest as it is an unchangeable identifier to the layer set of the current image build. Any change in it will lead to a diffent image and tags can be re-purposed/retaged.

## 4. Files

### Final `Dockerfile`
```
# syntax=docker/dockerfile:1.7

# ── build stage ──
FROM python:3.11-slim AS build
RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

# ── runtime stage ──
FROM python:3.11-slim AS runtime
COPY --from=build /opt/venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH

RUN <<EOF 
groupadd appmgt
useradd -g appmgt appmgt
mkdir /app
chown -R appmgt:appmgt /app
EOF

WORKDIR /app

COPY app.py .
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/healthz')" || exit 1

USER appmgt

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

### `Dockerfile.naive`
```
FROM ubuntu:latest

COPY . ./app

RUN apt update
RUN apt install -y python3-full && apt install python3-pip -y

RUN python3 -m venv /opt/venv && . /opt/venv/bin/activate

WORKDIR /app

RUN /opt/venv/bin/pip3 install -r requirements.txt
ENV STUDENT_NAME=Ike
EXPOSE 8080
CMD /opt/venv/bin/gunicorn --bind 0.0.0.0:8080 app:app
```

### `Dockerfile.secret`
```
# syntax=docker/dockerfile:1.7
FROM alpine:3.19
ARG TOKS
RUN --mount=type=secret,id=pypi_token \
    TOKEN=$(cat /run/secrets/pypi_token) && \
    echo "Token starts with ${TOKEN:0:4}" > /where-token-was-used
    
RUN echo "token is ${TOKS}" > /test-out
```

### `.dockerignore`
```
*.md

.gitignore
.git/

Dockerfile.*

.env*

__pycache__
*.pyc
```

## 5. Evidence

For each, paste the command and output. Trim long output to the relevant lines.

- `docker image ls cohort-greet` (all your tags from Part 4.2)
- `docker image history cohort-greet:multi` (truncate long base-image rows)
```
➜  assignment-03 git:(main) ✗ docker image history cohort-greet:multi 
IMAGE            CREATED        CREATED BY                                      SIZE      COMMENT
f7babed1753a     25 hours ago   CMD ["gunicorn" "--bind" "0.0.0.0:8080" "app…   0B        buildkit.dockerfile.v0
<missing>        25 hours ago   USER appmgt                                     0B        buildkit.dockerfile.v0
<missing>        25 hours ago   HEALTHCHECK &{["CMD-SHELL" "python -c \"impo…   0B        buildkit.dockerfile.v0
<missing>        25 hours ago   EXPOSE map[8080/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>        25 hours ago   COPY app.py . # buildkit                        360B      buildkit.dockerfile.v0
<missing>        25 hours ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>        25 hours ago   RUN /bin/sh -c groupadd appmgt
useradd -g ap…   4.36kB         buildkit.dockerfile.v0
<missing>        25 hours ago   ENV PATH=/opt/venv/bin:/usr/local/bin:/usr/l…   0B        buildkit.dockerfile.v0
<missing>        25 hours ago   COPY /opt/venv /opt/venv # buildkit             30.4MB    buildkit.dockerfile.v0
...
```
- The "no leak" / "LEAKED" check from Part 3.2
- `docker container run --rm hadolint/hadolint < Dockerfile`
```
➜  assignment-03 git:(main) ✗ docker container run --rm hadolint/hadolint < Dockerfile
➜  assignment-03 git:(main) ✗ hadolint Dockerfile 
```

- (Optional) URL of your pushed image

## 6. One trade-off I had to make

(2–4 sentences. Pick **one** decision where the slides offered multiple options and you had to choose: alpine vs slim vs distroless, USER 1000 vs `useradd app`, healthcheck via python vs installing curl, etc. Explain why you chose what you chose and what you'd give up by picking the other.)

While alpine and distroless offer smaller images and more security, the slim image makes up for it with maximum compatibility and ease of testing app workability with low coding knowledge.

Thinking now, Distroless should be used during runtime while slim should be used during build as this setup maximizes their strength and resuces their weaknesses drastically.

This would improve build/packaging compatibity and security while making sure debugging is possible during build stage as distroless possesses no shell for debugging suring runtime build.

## 7. One thing I'm still unsure about.

When to use git SHA and when to uses manifest/ID as there are both SHA pins.