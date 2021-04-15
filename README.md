# Docker tutorial

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

## Running image on a new instance

[Play with Docker](https://labs.play-with-docker.com/)

Do `docker run -dp 3000:3000 YOUR-USER-NAME/getting-started` in the sand box.

