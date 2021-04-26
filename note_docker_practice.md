# Notes - Docker Practice

[Docker Practice (in Chinese)](https://github.com/yeasy/docker_practice)

[Datawhale learning resources](https://github.com/datawhalechina/team-learning-program/tree/master/Docker)

## Image

### Dangling image `<none>`

```bash
docker image ls -f dangling=true
```

These images are generally no more useful and can be removed:

```bash
docker image prune
```
