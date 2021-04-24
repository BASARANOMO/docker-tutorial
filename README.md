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

    ```diff
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

### Starting ~~MySQL~~MariaDB

No available Docker image of `MySQL` under the Apple M1 ARM-64 environment. We use `MariaDB` instead.

Two ways to put a container on a network: (1) Assign it at start or (2) connect an existing container.

Create the network first and attach the ~~MySQL~~MariaDB container at startup:

1. Create the network:

    ```bash
    docker network create todo-app
    ```

2. Start a ~~MySQL~~MariaDB container and attach it to the network.

    ```diff
    docker run -d \
        --network todo-app --network-alias mysql \
        -v todo-mysql-data:/var/lib/mysql \
        -e MYSQL_ROOT_PASSWORD=secret \
        -e MYSQL_DATABASE=todos \
    -   mysql:5.7
    +   mariadb
    ```

3. To confirm we have the database up and running, connect to the database and verify it connects.

    ```diff
    -docker exec -it <mysql-container-id> mysql -p
    +docker exec -it <mysql-container-id> mariadb -p
    ```

    Password: `secret`

    Then in the ~~MySQL~~MariaDB shell, listh the databases and verify:

    ```diff sql
    -mysql> SHOW DATABASES;
    +MariaDB [(none)]> SHOW DATABASES;
    ```

### Connecting to ~~MySQL~~MariaDB

Make use of the [nicolaka/netshoot](https://github.com/nicolaka/netshoot) container, which ships with a lot of tools that are useful for troubleshooting or debugging networking issues.

1. Start a new container using the `nicolaka/netshoot` image.

    ```bash
    docker run -it --network todo-app nicolaka/netshoot
    ```

2. Inside the container, we're going to use the `dig` command, which is a useful DNS tool. We're going to look up the IP address for the hostname `mysql`

    ```bash
    dig mysql
    ```

    Output:

    ```
    ; <<>> DiG 9.16.11 <<>> mysql
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2315
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

    ;; QUESTION SECTION:
    ;mysql.				IN	A

    ;; ANSWER SECTION:
    mysql.			600	IN	A	172.18.0.2

    ;; Query time: 1 msec
    ;; SERVER: 127.0.0.11#53(127.0.0.11)
    ;; WHEN: Sat Apr 24 03:10:14 UTC 2021
    ;; MSG SIZE  rcvd: 44
    ```

    In the `ANSWER SECTION`, you will see an `A` record for `mysql` that resolves to `172.18.0.2`. While `mysql` isn't normally a valid hostname, Docker was able to resolve it to the IP address of the container that had that network alias (remember the `--network-alias` flag we used earlier?).

    What this means is... our app only simply needs to connect to a host named `mysql` and it'll talk to the database! It doesn't get much simpler than that!

### Running our App with ~~MySQL~~MariaDB

The todo app supports the setting of a few environment variables to specify ~~MySQL~~MariaDB connection settings. They are:

- `MYSQL_HOST` the hostname for the running ~~MySQL~~MariaDB server
- `MYSQL_USER` the username to use for the connection
- `MYSQL_PASSWORD` the password to use for the connection
- `MYSQL_DB` the database to use once connected

    #### Warning

        While using env vars to set connection settings is generally ok for development, it is HIGHLY DISCOURAGED when running applications in production. Diogo Monica, the former lead of security at Docker, wrote a fantastic blog post explaining why.

        A more secure mechanism is to use the secret support provided by your container orchestration framework. In most cases, these secrets are mounted as files in the running container. You'll see many apps (including the MySQL image and the todo app) also support env vars with a _FILE suffix to point to a file containing the variable.

        As an example, setting the MYSQL_PASSWORD_FILE var will cause the app to use the contents of the referenced file as the connection password. Docker doesn't do anything to support these env vars. Your app will need to know to look for the variable and get the file contents.