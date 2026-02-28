_Image_ : App, base OS, dependencies, etc...
_Container_ : A running image
_Volume_ : Persistent storage
_Dockerfile_ : IaC for an image.

Cheat Sheet
```bash
docker [command...] --help

# Some Commands
docker pull <image_name>     # pull an image 
docker images                # list images
docker ps [-a]               # check running containers. -a : all containers
docker stop <container_id>   # stop container gracefully
docker kill <container_id>   # stop container forcefully
docker rm <container_id>     # remove container
docker rm $(docker ps -a -q) # remove ALL containers
docker exec <container_id> <command>   # execute commands directly in container
docker exec -it <container_id> /bin/sh # -i : interactive shell , -t : tty

# Run an image/start container
docker run [--name name] [-d] [-p hostport:containerport] [-e VAR=thing]... [-v volume_name:path_in_container] [--network name] <namespace/name:tag>
	# --name : specify a container name
	# -d : detached mode... runs container in background
	# -p : port forwarding... so host can access container
	# -e : env variable... VAR1=thing
	# -v : mounts volume to container 
	# namespace/name : name of image (usually in the format `username/repo`)
	# tag : version
	# --network : specify a network to use... none = offline mode
	# --memory : limit memory available to container
	# --cpus : Limit CPU shares available to conteiner
	# EXAMPLES:
		docker run -d -p 8965:80 docker/getting-started:latest
		docker run -d -e NODE_ENV=development -e url=http://localhost:3001 -p 3001:2368 -v ghost-vol:/var/lib/ghost ghost

# Setting up persistent storage
docker volume create <name>   # create volume
docker volume ls              # verify
docker volume inspect <name>  # locate on local machine
docker volume rm <name>       # remove volume
docker volume prune           # remove unused volumes

# Creating bridge networks between containers
docker network create <name>  # Create a bridge
docker network ls             # Verify

# Building an image from a Dockerfile
vim Dockerfile                            # Setup Dockerfile
vim main.go                               # Example of setting up main go file.
docker build . -t <user/build>:[version]  # Build image. Modify version as needed.
docker run <name>                         # Validate it works

# Debugging
docker logs [-f] [--tail <num>] <container_id>  # View container logs
	# -f : realtime logs
	# --tail : last <> logs
docker stats <container_id> # Resource usage of container
docker top <container_id>   # Resource usage of processes inside a container

# Publishing to Docker Hub
compile binary (if using go)
docker build ...                              # Build
docker push <username>/<build_name>:[version] # Push to docker hub
docker image rm <username>/<build_name>       # Remove local image
docker pull <username>/<build_name>           # Pull from Docker Hub

# Versioning convention -- tag all new images w/ version num. AND "latest" tag
docker build -t bootdotdev/awesomeimage:5.4.6 -t bootdotdev/awesomeimage:latest .
docker push bootdotdev/awesomeimage --all-tags
```

#### Network Bridge & Load Balancer
Here is an example of setting up 2 webservers and a load balancer all within the same network using a network bridge and docker containers.

```bash
# Create bridge network
docker network create caddytest

# Create index files (vim <name>)
index1.html
<html>
  <body>
    <h1>Hello from server 1</h1>
  </body>
</html>

index2.html
<html>
  <body>
    <h1>Hello from server 1</h1>
  </body>
</html>

# Deploy 2 web servers (caddy)
docker run -d --name caddy1 --network caddytest -v $PWD/index1.html:/usr/share/caddy/index.html caddy
docker run -d --name caddy2 --network caddytest -v $PWD/index2.html:/usr/share/caddy/index.html caddy

# Validate
docker run -it --network caddytest docker/getting-started /bin/sh
curl caddy1
curl caddy2
exit

# Create Load Balancer file for caddy (vim Caddyfile)
localhost:80

reverse_proxy caddy1:80 caddy2:80 {
        lb_policy       round_robin
}

# Deploy Load Balancer
docker run -d --network caddytest -p 8880:80 -v $PWD/Caddyfile:/etc/caddy/Caddyfile caddy

# Validate... repeating this will show alternating web pages
curl http://localhost:8880/
```

#### The Bigger Picture

```
NOTE: Two companies rarely have identical processes. Instead of GitHub, it might be GitLab. Instead of Docker Hub, it might be ECR. Instead of Kubernetes, it might be Docker Swarm or a more managed service.
```

The Deployment Process
1. The developer (you) writes some new code
2. The developer commits the code to Git
3. The developer pushes a new branch to GitHub
4. The developer opens a [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests) to the `main` branch
5. A teammate reviews the PR and approves it (if it looks good)
6. The developer merges the pull request
7. Upon merging, an automated script, perhaps a [GitHub action](https://docs.github.com/en/actions), is started
8. The script builds the code (if it's a compiled language)
9. The script builds a new docker image with the latest program
10. The script pushes the new image to Docker Hub
11. The server that runs the containers, perhaps a [Kubernetes](https://kubernetes.io/) cluster, is told there is a new version
12. The k8s cluster pulls down the latest image
13. The k8s cluster shuts down old containers as it spins up new containers of the latest image

