# Notes - Docker Practice

[Docker Practice (in Chinese)](https://github.com/yeasy/docker_practice)

[Datawhale learning resources](https://github.com/datawhalechina/team-learning-program/tree/master/Docker)

## Image

### Dangling image `<none>`

```zsh
docker image ls -f dangling=true
```

These images are generally no more useful and can be removed:

```zsh
docker image prune
```

### Create an image

#### `Docker commit`

```zsh
docker run --name webserver -d -p 80:80 nginx
```

Access `http://localhost` and we'll see the welcome page of `nginx`.

If we want to cahnge the welcome page:

```zsh
docker exec -it webserver bash
root@30ca73daf89f:/# echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
root@30ca73daf89f:/# exit
exit
```

And then if we refresh the page, we'll see the page changed.

To check all changed files in our `webserver` container:

```zsh
docker diff webserver
```

We can then use the `docker commit` command to store all these changes and to update the image.

```zsh
docker commit \
    --author "BASARANOMO <your_email>" \
    --message "update the image" \
    webserver \
    nginx:v2
```

And we can run this new image:

```zsh
docker run --name web2 -d -p 81:80 nginx:v2
```

#### But be careful with `docker commit`

This command creates __black box image__ and is not recommanded (a lot of things could be changed and really difficult to be identified).

This problem comes from the fact that every time we do `docker commit`, we add a new layer on the top of the image. All modifications are restrained in this added layer without changing the previous ones, which makes the image get bloated and bloated.

### Use `Dockerfile`

```zsh
mkdir mynginx
cd mynginx
touch Dockerfile
```

And in the `Dockerfile` created, enter the commands:

```zsh
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

#### `FROM` command

`FROM` should always be the first command in a `Dockerfile`. It tells Docker which is the base image to use and to modify.

An empty image `scratch` is availble which means no base image used in this build (create an image from scratch):

```docker
FROM scratch
...
```

#### `RUN` command

- `shell` format: `RUN <command>`

    ```docker
    RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
    ```

- `exec` format: `RUN ["executable file", "arg1", "arg2"]`

An example of a `Dockerfile` with only `RUN` commands:

```docker
FROM debian:stretch

RUN apt-get update
RUN apt-get install -y gcc libc6-dev make wget
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```

The purpose of these commands is to install `redis`. Every `RUN` command creates a layer on the top of the image.

A better way to write this `Dockerfile`:

```docker
FROM debian:stretch

RUN set -x; buildDeps='gcc libc6-dev make wget' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

Seven layers are reduced to only one layer.

The `apt-get purge` command will be executed at the end to remove compile tools used previously.

#### Create a `nginx` image using `Dockerfile`

Run: `docker build -t nginx:v3 .`.

#### Context

The `.` in the command above represents the __context path__.

When we run `docker build` command, we build the image in the Docker's engine (on the server). In order to achieve this, Docker will package all the files in the context path and send to the Docker's engine (server).

For example, if we enter `COPY ./package.json /app/` in the `Dockerfile`, this is not to copy the `package.json` in the folder where we execute the `docker build` command, nor is it to copy the `package.json` in the folder where the `Dockerfile` is located, but to copy the `package.json` in the __context directory__.

The `COPY` command requires a _relative path_. So `COPY ../package.json /app` or `COPY /opt/xxxx /app` will never work (beyond the scope of the __context__).

By default (if not specified), we use the `Dockerfile` in the __context path__ when we run the `docker build` command.

But we can actually do like `-f ../Dockerfile.php` to specify the file to use as the `Dockerfile`.

#### Use Git repo to build

We can build with `URL`:

```zsh
docker build -t hello-world https://github.com/docker-library/hello-world.git#main:arm64v8/hello-world
```

Notes: You cannot specify the build-context directory (`arm64v8/hello-world` in the examples above) when using BuildKit as builder (`DOCKER_BUILDKIT=1`). Support for this feature is tracked in [buildkit#1684](https://github.com/moby/buildkit/issues/1684).


#### Use `tar`

```zsh
docker build http://server/context.tar.gz
```

#### Use standard input stream

```zsh
docker build - < Dockerfile
```

```zsh
cat Dockerfile | docker build -
```

```zsh
docker build - < context.tar.gz
```

### Other ways to create image

#### Import from rootfs

```zsh
docker import \
    http://download.openvz.org/template/precreated/ubuntu-16.04-x86_64.tar.gz \
    openvz/ubuntu:16.04
```

#### Export and import images using `docker save` and `docker load`

Not recommanded. It's better to use Docker Registry.

