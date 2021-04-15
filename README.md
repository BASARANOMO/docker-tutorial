# Docker tutorial

All infomation in this document comes from the [official Docker tutorial](https://github.com/docker/getting-started).

## Install

## Getting started - to do list app

## Sharing Docker image

### Create a repo

[Docker Hub](https://hub.docker.com/)

### Login to the Docker Hub

```bash
docker login -u YOUR-USER-NAME
```

### Give the image a tag

```bash
docker tag getting-started YOUR-USER-NAME/getting-started
```

### Docker push

```bash
docker push YOUR-USER-NAME/getting-started
```

### Running image on a new instance

[Play with Docker](https://labs.play-with-docker.com/)

Do `docker run -dp 3000:3000 YOUR-USER-NAME/getting-started` in the sand box.

## Persisting DB

### Seeing the container's filesystem in practice

1. Start a `Ubuntu` container that will create a file named /data.txt with a random number between 1 and 10000.

    ```bash
    docker run -d ubuntu bash -c "shuf -i 1-10000 -n 1 -o /data.txt && tail -f /dev/null"
    ```

2. Validate we can see the output by `exec`'ing into the container. To do so, open the Dashboard and click the first action of the container that is running the `Ubuntu` image.

    In the bash, run `cat /data.txt` to see the content of the `/data.txt` file.

    Or in the command line, do `docker exec <container-id> cat /data.txt`.

3. Now start another `Ubuntu` container:

    ```bash
    docker run -it ubuntu ls /
    ```

    No `/data.txt` in the filesystem.

4. Remove the first container using `docker rm -f`.

