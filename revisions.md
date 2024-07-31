build option already has a default value so using ``` build: . ``` is just simplifying if you have alternative docker-files for you build, then you can specifiy the dockerfile with

```
build:
    context: ./
    dockerfile: alternate_docker_file
```


