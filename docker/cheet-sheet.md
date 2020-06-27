`docker run --rm -it ${image} /bin/bash`

## Stop all containers and delete all images

After this commands every image and container has to be pulled again and created. Clean reset if the hard drive is messed up.

```bash
docker stop $(docker ps -a -q)
docker rm -v $(docker ps -a -q)
docker rmi $(docker images -q)
```