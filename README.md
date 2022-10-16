# **Installing Project Dependencies**
Before perform the steps bellow, remember to enable the HyperV Virtualization in your BIOS.

## **Installing Docker**
### **Set up the repository**
You can follow these steps to install Docker in [Windows](https://docs.docker.com/desktop/install/windows-install/) or [Linux](https://docs.docker.com/engine/install/ubuntu/). If you're using Ubuntu OS to build the environment, run the commands below.

1 - Update the apt package index and install packages to allow apt to use a repository over HTTPS:
```
$ sudo apt-get update
```
```
$ sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
2 - Add Docker’s official GPG key and set up the repository
```
$ sudo mkdir -p /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
    https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
### **Install Docker Engine**

1 - Update the apt package index, and install the latest version of Docker Engine, containerd, and Docker Compose, or go to the next step to install a specific version:
```
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

2 - Verify that Docker Engine is installed correctly by running the hello-world image
```
$ sudo service docker start
$ sudo docker run hello-world
```

## **Installing Minikube**
Minikube is Single-Node Kubernetes Cluster, focusing on making it easy to learn and develop for Kubernetes. Follow the steps in the guide [how-to-install](https://minikube.sigs.k8s.io/docs/start/)...

...Or run these commands in your Ubuntu terminal.
```
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
$ sudo install minikube-linux-amd64 /usr/local/bin/minikube
$ rm minikube-linux-amd64.deb
```


## **Installing Kubectl**
Kubectl is a CLI tool used to manage Kubernete Clusters.
Download the latest release running
```
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ mv kubectl ~/home
``` 
Or if you are a Windows user, [try it](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/).


# **Building Environment**
The plan is to build a Infrastructure with a Nginx docker container serving as a Revert Proxy to Kubernetes Cluster, and exposing the Proxy Server to WWW.  This way, the Cluster can be managed from anywhere, making possible the comunication with Github CI/CD Runners. 

The access to the Cluster needs a Basic Authentication throught Nginx server, passed thourgh the header request when integrating with kubectl.

## Creating a virtual network
As we're interested to start up two Docker Containers, one with a Nginx image and the other running a Minikube Cluster, both needs to comunicate with each other, to do so, the containers must be in the same network.
Let's create a virtual network using `docker network create` command with `--subnet=CIDR_BLOCK` and `--gateway=GW_ADDRESS` flags.
```
$ docker network create -d bridge my-net --subnet=10.1.0.0/24 --gateway=10.1.0.1
```
The `-d` option is used to set the network manage driver `bridge` or `overlay`.

## Starting Minikube Kubernetes Single-Node Cluster
Before starts Nginx container, build the minikube Cluster to get the first available IP Address in network, otherwise an error are raised signaling overlapping IPs.

After create `my-net` network, we can start minikube using `--network` option to set the virtual network driver.
```
$ minikube start --network=my-net --driver=docker
```


## Seting Up Nginx Reverse Proxy
Nginx by default uses configuration files located in `/etc/nginx/conf.d/` to starts a Web Server.
Every file with .conf extension

Run the command below to start up nginx server:
```
$ docker run -d \
    --name nginx \             
    -p 8080:80 \
    -v ~/.minikube/profiles/minikube/client.key:/etc/nginx/certs/minikube-client.key \
    -v ~/.minikube/profiles/minikube/client.crt:/etc/nginx/certs/minikube-client.crt \
    -v /etc/nginx/conf.d/:/etc/nginx/conf.d \
    -v /etc/nginx/.htpasswd:/etc/nginx/.htpasswd \
    --network=my-net nginx
```

