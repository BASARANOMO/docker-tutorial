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

