# Swarm Cluster

This documentation cover the configuration for a docker swarm cluster.

This cofiguration cover the following functionalities:

## Before Start

* Firewall rules for Docker daemons using overlay networks

    You need the following ports open to traffic to and from each Docker host participating on an overlay network:

    * TCP port 2377 for cluster management communications
    * TCP and UDP port 7946 for communication among nodes
    * UDP port 4789 for overlay network traffic

[Overlay Networks Documentation](https://docs.docker.com/network/overlay/)

## Traefik

1) Handle connections.
2) Expose specific services and applications based on their domain names.
3) Handle multiple domains.
4) Acquire and renew HTTPS certificates automatically with [Let's Encrypt](https://letsencrypt.org/).
5) Add HTTP Basic Auth for any service that you need to protect and doesn't have it own security.
6) Get all its configurations automatically from Docker labels set in your stacks.

## Portainer

Portainer is a lightweight management UI which allows you to easily manage your different Docker environments.  
Portainer allows you to manage all your Docker resources (containers, images, volumes, networks and more) ! It is compatible with the standalone Docker engine and with Docker Swarm mode.

# Configuration

## Start Docker Swarm Mode

First we need to initialize the swarm mode using `docker swarm init`. If the system has multiple IP address, `--advertise-addr` must be specified with the correct address for inter-manager communication and overlay networking.

[Docker swarm init documentation](https://docs.docker.com/engine/reference/commandline/swarm_init/)

## Workers and Managers

For add a manager or worker to a swarm cluster you need a join token, you can easily acquire a token running one of the following commands on a manager node:

**Worker Node:** 
>`docker swarm join-token worker`  

**Manager Node:** 
>`docker swarm join-token manager`

These commands will prompt a output like:

> To add a worker to this swarm, run the following command:
>
> docker swarm join --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c 192.168.99.100:2377

Run the command from the output on the new node to join the swarm.  

[Join nodes Documentation](https://docs.docker.com/engine/swarm/join-nodes)

By default manager nodes also act as a worker nodes. This means the scheduler can assign tasks to a manager node.  
However, because manager nodes use the Raft consensus algorithm to replicate data in a consistent way, they are sensitive to resource starvation. You should isolate managers in your swarm from processes that might block swarm operations like swarm heartbeat or leader elections.

To avoid interference with manager node operation, you can drain manager nodes to make them unavailable as worker nodes:

>`docker node update --availability drain <NODE>`

When you drain a node, the scheduler reassigns any tasks running on the node to other available worker nodes in the swarm. It also prevents the scheduler from assigning tasks to the node.

[Docker Swarm Admin Documentation](https://docs.docker.com/engine/swarm/admin_guide/)

## Deploy Traefik Stack

* Create a network that will be shared with Traefik and the containers that should be accessible from the outside, with:

>`docker network create --driver=overlay traefik-public`

* Create an environment variable with your email, to be used for the generation of Let's Encrypt certificates:

> `export EMAIL=admin@example.com`

* Create an environment variable with the domain you want to use for the Traefik UI (user interface) and the Consul UI of the host, e.g.:

>`export DOMAIN=sys.example.com`

You will access the Traefik UI at traefik.`<your domain>`, e.g. `traefik.sys.example.com` and the Consul UI at consul.`<your domain>`, e.g. `consul.sys.example.com`.

So, make sure that your DNS records point traefik.`<your domain>` and consul.`<your domain>` to one of the IPs of the cluster.

* Create an environment variable with a username (you will use it for the HTTP Basic Auth for Traefik and Consul UIs), for example:

>`export USERNAME=admin`

* Create an environment variable with the password, e.g.:

>`export PASSWORD=changethis`

Use openssl to generate the "hashed" version of the password and store it in an environment variable:

>`export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)`

**Deploy IT**

Deploy the stack with:
>`docker stack deploy -c traefik-host.yml traefik-consul`

[Traefik Stack Documentation](https://github.com/tiangolo/blog-posts/tree/master/docker-swarm-mode-and-distributed-traefik-proxy-with-https)


## Deploy Portainer Stack

The portainer.yml is a docker-compose file with the configurations needed to work properly with Traefik proxy. Once deployed the Traefik will generate the Let's Encrypt certificate.

You will acess the Portainer UI at portainer.`<your domain>`, e.g `portainer.sys.example.com`.

So, make sure that your DNS records point portainer.`<your domain>` to one of the IPs of the cluster.

Deploy the stack with:
>`docker stack deploy -c portainer.yml portainer`

[Portainer Documentation](https://portainer.readthedocs.io/en/latest/deployment.html#declaring-the-docker-environment-to-manage-upon-deployment)

## Deploy Swarmprom Stack (Monitoring and Alert)

Using Portainer UI navigate to `Stacks > Add stack`. At `name` field type `swarmprom` and choose the `git Repository` option and fill the following fields.

**Repository URL:**
>`https://github.com/stefanprodan/swarmprom`

**Repository reference:**
>`refs/heads/master`

**Compose path:**
>`docker-compose.traefik.yml`

At `Environment variables` section add the following variables:

>`ADMIN_USER=admin`  
`ADMIN_PASSWORD=changethis`  
`DOMAIN=sys.example.com`  
`HASHED_PASSWORD=<output of openssl passwd -apr1 $PASSWORD>`

[Swarmprom Documentation](https://github.com/stefanprodan/swarmprom)