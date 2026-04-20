_Kubernetes_ : Open-source system for automating deployment, scaling, and management of containerized applications
	Manages resources and allocates them to applications that are running on it.
	Configured using YAML files.
_Kubectl_ : Kubernetes command-line tool
_Minikube_ : Tool to run single-node k8s cluster on local machine. For learning, not for prod!
_Pods_ : The smallest deployable units of computing that you can create and manage in k8s.
	Represents one or more running container(s) in a cluster. Just wrappers around containers.
	Ephemeral. Designed to be spun up, torn down, restarted, etc...
_Deployments_ : Provides declarative updates for Pods and ReplicaSets
	Deployment Controller's job is to make current state match desired state.
	The reason why deleting a pod spawns a new one.
	A higher-level abstraction that manages ReplicaSets for you.
_ReplicaSet_ : Maintains a stable set of replica Pods running at any given time.
	Will probably never use ReplicaSets directly.

Cheat Sheet
```bash
# Install
https://kubernetes.io/docs/tasks/tools/   # kubectl install
https://minikube.sigs.k8s.io/docs/start/  # minikube install

# Setup
minikube start --extra-config="apiserver.cors-allowed-origins=['http://boot.dev']"
minikube dashboard --port=63840 # dashboard setup

# Deploy an image
kubectl create deployment <name> --image=<URL_to_docker_image>  # Deploy
kubectl get deployments    # Verify

# Accessing the Web page
kubectl get pods                            # Annotate pod name
kubectl port-forward <podname> 8080:8080    # Port Forward
	http://localhost:8080                   # Access in browser

# Adding more instances of a pod
kubectl get pods               # List pods. IGNORE last section of podname 
kubectl edit deployment <name> # under "spec" section, modify "replicas" field
kubectl get pods               # Verify

# More Pods
kubectl logs <podname>         # Print logs of pod
kubectl delete pod <podname>   # Kills pod

# Pod Status
kubectl get pods [-o wide]     # List pods and status. -o : more info
kubectl proxy                  # Spins up proxy server. Visit URL for pod info
	http://127.0.0.1:8001/api/v1/namespaces/default/pods

# Deployments
kubectl get deployment <name> -o yaml  # Check current deployment YAML
kubectl edit deployment <name>         # Edit deployment config in place
kubectl get pod                        # Check pods
kubectl proxy                          # visit url for pod info
kubectl get replicasets                # Get ReplicaSets

# Modifying a deployment locally and applying it
kubectl get deployment <name> -o yaml > web-deployment.yaml  # Output locally
vim web-deployment.yaml                                      # modify
kubectl apply -f web-deployment.yaml                         # Apply
	# Repeat these steps 2x to resolve error.

```