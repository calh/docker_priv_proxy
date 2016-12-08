# Docker Privileged Proxy

This image is a wrapper / proxy intended to workaround the limitation
of --cap-add in Docker Swarm Mode services.  (Which doesn't exist yet,
 watch [this GH issue](https://github.com/docker/docker/issues/24862) 
if you're interested )

Many images need `--privileged` mode, or some form of it using 
parameters like `--device`, `--ipc`, `--net`, or `--cap-add`.

This image is a very simple debian image that binds the docker socket
from the host, then fires up a container using `docker run`.  

socat is used to proxy the privileged container's port using a swarm overlay network.

NOTE:  This requires docker engine 1.13 for attachable networks, and is currently in testing.
This could hypothetically be replaced with --link on older Docker versions, but 
I haven't cared to do this.

## Building

`docker build -t docker_priv_proxy .`

## Example Usage

The primary reason I needed this functionality was for DB2 testing/staging setups.  
Here's a real live demo you can try out.

First, create an overlay network with --attachable so that the container
created with `docker run` can be accessed from the proxy service

`docker network create -d overlay --attachable --subnet 10.1.1.0/24 db2_net`

Next, create the `db2_proxy` swarm service.  The `DOCKER_RUN` variable
can have any parameters that `docker run` would accept.  

```
docker service create \
  --name db2_proxy \
  --network db2_net \
  --mount "type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock" \
  -e DOCKER_RUN="--network db2_net --cap-add=IPC_OWNER --ipc=host -e DB2INST1_PASSWORD=password -e LICENSE=accept ibmcom/db2express-c:latest db2start" \
  -e PORT=50000 \
  -p 50002:50000 \
  docker_priv_proxy
```

The `db2_proxy` service will start up and publish the external 50002 port into the proxy's port 50000.
The proxy will create a new db2 container on `db2_net` and proxy port 50000 to it.


