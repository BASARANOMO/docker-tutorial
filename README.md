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

### Container volumes

[Docker's official doc about volumes](https://docs.docker.com/storage/volumes/)

"Volumes provide the ability to connect specific filesystem paths of the container back to the host machine. If a directory in the container is mounted, changes in that directory are also seen on the host machine. If we mount that same directory across container restarts, we'd see the same files."

### Example of using __named volumes__

Named volume: a bucket of data stored on the host

1. Create a volume using the `docker volume create` command:

   ```bash
   docker volume create todo-db
   ```

2. Restart the todo app container:

    ```bash
    docker run -dp 3000:3000 -v todo-db:/etc/todos getting-started
    ```

3. Add some new items then remove the container

    ```bash
    docker rm -f <id>
    ```

4. Start a new container

### Diving into our volume

Question: where is Docker actually storing the data when using a named volume?

Use:

```bash
docker volume inspect todo-db
```

The `Mountpoint` shown is the actual location on the disk where the data is stored. (Root access is necessary to access this directory from the host.)

## Using bind mounts

__Bind mounts__: the other way to persist data.

Often used to provide additional data into containers.

When working on an application, we can use a bind mount to mount our source code into the container to let it see code changes, respond, and let us see the changes right away.

### Quick volume type comparison

| | Named volumes | Bind mounts |
|--|--|--|
| Host location | Docker chooses | You control |
| Mount example (using `-v`) | my-volume:/usr/local/data | /path/to/data:/usr/local/data |
| Populates new volume with container contents | Yes | No |
| Support volume drivers | Yes | No |

### Starting a dev-mode container

- Mount our source code into the container
- Install all dependencies, including the "dev" dependencies
- Start `nodemon` to watch for filesystem changes

1. Shutdown all previous `getting-started` containers.

2. Run the command below in `/app` folder:

    ```bash
    docker run -dp 3000:3000 \
        -w /app -v "$(pwd):/app" \
        node:12-alpine \
        sh -c "apk add python && apk add g++ && apk add make && yarn install && yarn run dev"
    ```

3. Make a change to the app:

    ```git
    -                         {submitting ? 'Adding...' : 'Add Item'}
    +                         {submitting ? 'Adding...' : 'Add'}
    ```

4. Build:

   ```bash
   docker build -t getting-started .
   ```

## Multi-container apps

A few reasons to use multi-containers:

- There's a good chance you'd have to scale APIs and front-ends differently than databases
- Separate containers let you version and update versions in isolation
- While you may use a container for the database locally, you may want to use a managed service for the database in production. You don't want to ship your database engine with your app then.
- Running multiple processes will require a process manager (the container only starts one process), which adds complexity to container startup/shutdown.

### Container networking

Rule:

> __If two containers are on the same network, they can talk to each other. If they aren't, they can't.__

### Starting MySQL

