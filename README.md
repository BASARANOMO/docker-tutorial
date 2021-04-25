# Docker tutorial

All infomation in this document comes from the [official Docker tutorial](https://github.com/docker/getting-started) and __adapted to MacOS 11 and ARM-64 architecture__.

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
    - docker exec -it <mysql-container-id> mysql -p
    + docker exec -it <mysql-container-id> mariadb -p
    ```

    Password: `secret`

    Then in the ~~MySQL~~MariaDB shell, listh the databases and verify:

    ```diff sql
    - mysql> SHOW DATABASES;
    + MariaDB [(none)]> SHOW DATABASES;
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

1. We'll specify each of the environment variables above, as well as connect the container to our app network.

    ```bash
    docker run -dp 3000:3000 \
        -w /app -v "$(pwd):/app" \
        --network todo-app \
        -e MYSQL_HOST=mysql \
        -e MYSQL_USER=root \
        -e MYSQL_PASSWORD=secret \
        -e MYSQL_DB=todos \
        node:12-alpine \
        sh -c "yarn install && yarn run dev"
    ```

2. If we look at the logs for the container (`docker logs <container-id>`), we should see a message indicating it's using the mysql database.

3. Open the app in your browser and add a few items to your todo list.

4. Connect to the ~~mysql~~MariaDB database and prove that the items are being written to the database. Remember, the password is `secret`.

    ```diff
    - docker exec -it <mysql-container-id> mysql -p
    + docker exec -it <mysql-container-id> mariadb -p
    ```

    And in the mysql shell, run the following:

    ```sql
    MariaDB [todos]> select * from todo_items;
    ```

    We'll see:

    ```
    +--------------------------------------+--------+-----------+
    | id                                   | name   | completed |
    +--------------------------------------+--------+-----------+
    | e69348d3-636f-4e26-bd33-fb7f5bc34167 | test   |         0 |
    | 7c341428-d6a2-4902-a8a5-7bf08668f77d | hi     |         0 |
    | 020cc3d5-4e5b-41fa-83f7-97ea659a324a | strike |         0 |
    +--------------------------------------+--------+-----------+
    3 rows in set (0.000 sec)
    ```

    Obviously, your table will look different because it has your items. But, you should see them stored there!

## Using Docker Compose

[Docker Compose](https://docs.docker.com/compose/) is a tool that was developed to help define and share multi-container applications. With Compose, we can create a YAML file to define the services and with a single command, can spin everything up or tear it all down.

Advantage of using Compose: you can define your application stack in a file, keep it at the root of your project repo (it's now version controlled), and easily enable someone else to contribute to your project.

### Installing Docker Compose

Use the command to check Docker Compose's version:

```bash
docker-compose version
```

### Create our Compose file

1. At the root of the app project, create a file named `docker-compose.yml`.

2. In the compose file, we'll start off by defining the schema version. In most cases, it's best to use the latest supported version. You can look at the [Compose file reference](https://docs.docker.com/compose/compose-file/) for the current schema versions and the compability matrix.

3. Next, we'll define the list of services (or containers) we want to run as part of our application.

And now, we'll start migrating a service at a time into the compose file.

### Define the App service

To remember, this was the command we were using to define our app container.

```bash
docker run -dp 3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:12-alpine \
  sh -c "yarn install && yarn run dev"
```

1. First, let's define the service entry and the image for the container. We can pick any name for the service. The name will automatically become a network alias, which will be useful when defining our MySQL service.

    ```
    version: "3.7"

    services:
      app:
        image: node:12-alpine
    ```

2. Typically, you will see the command close to the `image` definition, although there is no requirement on ordering. So, let's go ahead and move that into our file.

    ```
    version: "3.7"

    services:
      app:
        image: node:12-alpine
        command: sh -c "yarn install && yarn run dev"
    ```

3. Let's migrate the `-p 3000:3000` part of the command by defining the `ports` for the service. We will use the short syntax here, but there is also a more verbose long syntax available as well.

    ```
    version: "3.7"

    services:
      app:
        image: node:12-alpine
        command: sh -c "yarn install && yarn run dev"
        ports:
           - 3000:3000
    ```

4. Next, we'll migrate both the working directory (`-w /app`) and the volume mapping (`-v "$(pwd):/app"`) by using the `working_dir` and `volumes` definitions. Volumes also has a short and long syntax.

    One advantage of Docker Compose volume definitions is we can use relative paths from the current directory.

    ```
    version: "3.7"

    services:
      app:
        image: node:12-alpine
        command: sh -c "yarn install && yarn run dev"
        ports:
           - 3000:3000
        working_dir: /app
        volumes:
           - ./:/app
    ```

5. Finally, we need to migrate the environment variable definitions using the `environment` key.

    ```
    version: "3.7"

    services:
      app:
        image: node:12-alpine
        command: sh -c "yarn install && yarn run dev"
        ports:
           - 3000:3000
        working_dir: /app
        volumes:
           - ./:/app
        environment:
            MYSQL_HOST: mysql
            MYSQL_USER: root
            MYSQL_PASSWORD: secret
            MYSQL_DB: todos
    ```

### Defining the ~~MySQL~~MariaDB service

Now, it's time to define the ~~MySQL~~MariaDB service. The command that we used for that container was the following:

```bash
docker run -d \
  --network todo-app --network-alias mysql \
  -v todo-mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=todos \
  mariadb
```

1. We will first define the new service and name it `mysql` so it automatically gets the network alias. We'll go ahead and specify the image to use as well.

    ```diff
    version: "3.7"

    services:
      app:
        # The app service definition
    mysql:
    -   image: mysql:5.7
    +   image: mariadb
    ```

2. Next, we'll define the volume mapping. When we ran the container with `docker run`, the named volume was created automatically. However, that doesn't happen when running with Compose. We need to define the volume in the top-level `volumes:` section and then specify the mountpoint in the service config. By simply providing only the volume name, the default options are used. There are many more options available though.

    ```
    version: "3.7"

    services:
      app:
        # The app service definition
      mysql:
        image: mysql:5.7
        volumes:
          - todo-mysql-data:/var/lib/mysql

    volumes:
      todo-mysql-data:
    ```

3. Finally, we only need to specify the environment variables.

    ```
    version: "3.7"

    services:
      app:
        # The app service definition
      mysql:
        image: mysql:5.7
        volumes:
          - todo-mysql-data:/var/lib/mysql
        environment: 
          MYSQL_ROOT_PASSWORD: secret
          MYSQL_DATABASE: todos

    volumes:
      todo-mysql-data:
    ```

At this point, our complete docker-compose.yml should look like this:

```
version: "3.7"

services:
  app:
    image: node:12-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:5.7
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment: 
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

### Running our application stack

Now that we have our `docker-compose.yml` file, we can start it up!

1. Make sure no other copies of the app/db are running first (`docker ps` and `docker rm -f <ids>`).

2. Start up the application stack using the `docker-compose` up command. We'll add the `-d` flag to run everything in the background.

    ```bash
    docker-compose up -d
    ```

    When we run this, we should see output like this:

    ```
    Creating network "app_default" with the default driver
    Creating volume "app_todo-mysql-data" with default driver
    Creating app_app_1   ... done
    Creating app_mysql_1 ... done
    ```

    You'll notice that the volume was created as well as a network! By default, Docker Compose automatically creates a network specifically for the application stack (which is why we didn't define one in the compose file).

3. Let's look at the logs using the `docker-compose logs -f` command. You'll see the logs from each of the services interleaved into a single stream. This is incredibly useful when you want to watch for timing-related issues. The `-f` flag "follows" the log, so will give you live output as it's generated.

    If you don't already, you'll see output that looks like this...

    ```
    Attaching to app_app_1, app_mysql_1
    mysql_1  | 2021-04-25 07:29:13+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 1:10.5.9+maria~focal started.
    mysql_1  | 2021-04-25 07:29:13+00:00 [Note] [Entrypoint]: Switching to dedicated user 'mysql'
    mysql_1  | 2021-04-25 07:29:13+00:00 [Note] [Entrypoint]: Entrypoint script for MySQL Server 1:10.5.9+maria~focal started.
    mysql_1  | 2021-04-25 07:29:13+00:00 [Note] [Entrypoint]: Initializing database files
    mysql_1  | 
    mysql_1  | 
    mysql_1  | PLEASE REMEMBER TO SET A PASSWORD FOR THE MariaDB root USER !
    mysql_1  | To do so, start the server, then issue the following commands:
    mysql_1  | 
    mysql_1  | '/usr/bin/mysqladmin' -u root password 'new-password'
    mysql_1  | '/usr/bin/mysqladmin' -u root -h  password 'new-password'
    mysql_1  | 
    mysql_1  | Alternatively you can run:
    mysql_1  | '/usr/bin/mysql_secure_installation'
    mysql_1  | 
    mysql_1  | which will also give you the option of removing the test
    mysql_1  | databases and anonymous user created by default.  This is
    mysql_1  | strongly recommended for production servers.
    mysql_1  | 
    mysql_1  | See the MariaDB Knowledgebase at https://mariadb.com/kb or the
    mysql_1  | MySQL manual for more instructions.
    mysql_1  | 
    mysql_1  | Please report any problems at https://mariadb.org/jira
    mysql_1  | 
    mysql_1  | The latest information about MariaDB is available at https://mariadb.org/.
    mysql_1  | You can find additional information about the MySQL part at:
    mysql_1  | https://dev.mysql.com
    mysql_1  | Consider joining MariaDB's strong and vibrant community:
    mysql_1  | https://mariadb.org/get-involved/
    mysql_1  | 
    mysql_1  | 2021-04-25 07:29:13+00:00 [Note] [Entrypoint]: Database files initialized
    mysql_1  | 2021-04-25 07:29:13+00:00 [Note] [Entrypoint]: Starting temporary server
    mysql_1  | 2021-04-25 07:29:13+00:00 [Note] [Entrypoint]: Waiting for server startup
    mysql_1  | 2021-04-25  7:29:13 0 [Note] mysqld (mysqld 10.5.9-MariaDB-1:10.5.9+maria~focal) starting as process 105 ...
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Uses event mutexes
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Compressed tables use zlib 1.2.11
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Number of pools: 1
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Using ARMv8 crc32 + pmull instructions
    mysql_1  | 2021-04-25  7:29:13 0 [Note] mysqld: O_TMPFILE is not supported on /tmp (disabling future attempts)
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Using Linux native AIO
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Initializing buffer pool, total size = 134217728, chunk size = 134217728
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Completed initialization of buffer pool
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: If the mysqld execution user is authorized, page cleaner thread priority can be changed. See the man page of setpriority().
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: 128 rollback segments are active.
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Creating shared tablespace for temporary tables
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: File './ibtmp1' size is now 12 MB.
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: 10.5.9 started; log sequence number 45118; transaction id 20
    mysql_1  | 2021-04-25  7:29:13 0 [Note] Plugin 'FEEDBACK' is disabled.
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Loading buffer pool(s) from /var/lib/mysql/ib_buffer_pool
    mysql_1  | 2021-04-25  7:29:13 0 [Note] InnoDB: Buffer pool(s) load completed at 210425  7:29:13
    mysql_1  | 2021-04-25  7:29:13 0 [Warning] 'user' entry 'root@5c9ddc2a2b02' ignored in --skip-name-resolve mode.
    mysql_1  | 2021-04-25  7:29:13 0 [Warning] 'proxies_priv' entry '@% root@5c9ddc2a2b02' ignored in --skip-name-resolve mode.
    mysql_1  | 2021-04-25  7:29:13 0 [Note] Reading of all Master_info entries succeeded
    mysql_1  | 2021-04-25  7:29:13 0 [Note] Added new Master_info '' to hash table
    mysql_1  | 2021-04-25  7:29:13 0 [Note] mysqld: ready for connections.
    mysql_1  | Version: '10.5.9-MariaDB-1:10.5.9+maria~focal'  socket: '/run/mysqld/mysqld.sock'  port: 0  mariadb.org binary distribution
    mysql_1  | 2021-04-25 07:29:14+00:00 [Note] [Entrypoint]: Temporary server started.
    mysql_1  | Warning: Unable to load '/usr/share/zoneinfo/leap-seconds.list' as time zone. Skipping it.
    mysql_1  | Warning: Unable to load '/usr/share/zoneinfo/leapseconds' as time zone. Skipping it.
    mysql_1  | Warning: Unable to load '/usr/share/zoneinfo/tzdata.zi' as time zone. Skipping it.
    mysql_1  | 2021-04-25  7:29:16 5 [Warning] 'proxies_priv' entry '@% root@5c9ddc2a2b02' ignored in --skip-name-resolve mode.
    mysql_1  | 2021-04-25 07:29:16+00:00 [Note] [Entrypoint]: Creating database todos
    mysql_1  | 
    mysql_1  | 2021-04-25 07:29:16+00:00 [Note] [Entrypoint]: Stopping temporary server
    mysql_1  | 2021-04-25  7:29:16 0 [Note] mysqld (initiated by: root[root] @ localhost []): Normal shutdown
    mysql_1  | 2021-04-25  7:29:16 0 [Note] Event Scheduler: Purging the queue. 0 events
    mysql_1  | 2021-04-25  7:29:16 0 [Note] InnoDB: FTS optimize thread exiting.
    mysql_1  | 2021-04-25  7:29:16 0 [Note] InnoDB: Starting shutdown...
    mysql_1  | 2021-04-25  7:29:16 0 [Note] InnoDB: Dumping buffer pool(s) to /var/lib/mysql/ib_buffer_pool
    mysql_1  | 2021-04-25  7:29:16 0 [Note] InnoDB: Buffer pool(s) dump completed at 210425  7:29:16
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Removed temporary tablespace data file: "ibtmp1"
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Shutdown completed; log sequence number 45130; transaction id 21
    mysql_1  | 2021-04-25  7:29:17 0 [Note] mysqld: Shutdown complete
    mysql_1  | 
    mysql_1  | 2021-04-25 07:29:17+00:00 [Note] [Entrypoint]: Temporary server stopped
    mysql_1  | 
    mysql_1  | 2021-04-25 07:29:17+00:00 [Note] [Entrypoint]: MySQL init process done. Ready for start up.
    mysql_1  | 
    mysql_1  | 2021-04-25  7:29:17 0 [Note] mysqld (mysqld 10.5.9-MariaDB-1:10.5.9+maria~focal) starting as process 1 ...
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Uses event mutexes
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Compressed tables use zlib 1.2.11
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Number of pools: 1
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Using ARMv8 crc32 + pmull instructions
    mysql_1  | 2021-04-25  7:29:17 0 [Note] mysqld: O_TMPFILE is not supported on /tmp (disabling future attempts)
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Using Linux native AIO
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Initializing buffer pool, total size = 134217728, chunk size = 134217728
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Completed initialization of buffer pool
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: If the mysqld execution user is authorized, page cleaner thread priority can be changed. See the man page of setpriority().
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: 128 rollback segments are active.
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Creating shared tablespace for temporary tables
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full; Please wait ...
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: File './ibtmp1' size is now 12 MB.
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: 10.5.9 started; log sequence number 45130; transaction id 20
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Loading buffer pool(s) from /var/lib/mysql/ib_buffer_pool
    mysql_1  | 2021-04-25  7:29:17 0 [Note] Plugin 'FEEDBACK' is disabled.
    mysql_1  | 2021-04-25  7:29:17 0 [Note] InnoDB: Buffer pool(s) load completed at 210425  7:29:17
    mysql_1  | 2021-04-25  7:29:17 0 [Note] Server socket created on IP: '::'.
    mysql_1  | 2021-04-25  7:29:17 0 [Warning] 'proxies_priv' entry '@% root@5c9ddc2a2b02' ignored in --skip-name-resolve mode.
    mysql_1  | 2021-04-25  7:29:17 0 [Note] Reading of all Master_info entries succeeded
    mysql_1  | 2021-04-25  7:29:17 0 [Note] Added new Master_info '' to hash table
    mysql_1  | 2021-04-25  7:29:17 0 [Note] mysqld: ready for connections.
    mysql_1  | Version: '10.5.9-MariaDB-1:10.5.9+maria~focal'  socket: '/run/mysqld/mysqld.sock'  port: 3306  mariadb.org binary distribution
    mysql_1  | 2021-04-25  7:29:18 3 [Warning] Aborted connection 3 to db: 'unconnected' user: 'unauthenticated' host: '172.19.0.2' (This connection closed normally without authentication)
    app_1    | yarn install v1.22.5
    app_1    | [1/4] Resolving packages...
    app_1    | success Already up-to-date.
    app_1    | Done in 0.19s.
    app_1    | yarn run v1.22.5
    app_1    | $ nodemon src/index.js
    app_1    | [nodemon] 1.19.2
    app_1    | [nodemon] to restart at any time, enter `rs`
    app_1    | [nodemon] watching dir(s): *.*
    app_1    | [nodemon] starting `node src/index.js`
    app_1    | Waiting for mysql:3306..
    app_1    | Connected!
    app_1    | Connected to mysql db at host mysql
    app_1    | Listening on port 3000
    ```
    
    The service name is displayed at the beginning of the line (often colored) to help distinguish messages. If you want to view the logs for a specific service, you can add the service name to the end of the logs command (for example, `docker-compose logs -f app`).

4. At this point, you should be able to open your app and see it running. And hey! We're down to a single command!

### Seeing our App Stack in Docker Dashboard

### Tearing it All Down

When you're ready to tear it all down, simply run `docker-compose down` or hit the trash can on the Docker Dashboard for the entire app. The containers will stop and the network will be removed.
