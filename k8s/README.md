# Install Docker

## Official Webiste steps to install
[Official Docker Installation](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

## add an user to docker group
```
    sudo groupadd docker
    sudo usermod -aG docker $USER
```

# Install Minikube

## install kubectl and virtualbox
```
    apt-get update && apt-get install -y apt-transport-https curl gnupg2
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | tee -a /etc/apt/sources.list.d/kubernetes.list
    wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | apt-key add -
    wget -q https://www.virtualbox.org/download/oracle_vbox.asc -O- | apt-key add -
    add-apt-repository "deb http://download.virtualbox.org/virtualbox/debian xenial contrib"
    apt-get update
    apt-get install -y kubectl virtualbox-6.0
```

### donwload minikube and add it to bin
```
    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64   && chmod +x minikube
    cp minikube /usr/local/bin && rm minikube
```

### start minikube
```
    minikube start --cpus 4 --memory 8192
```

### enable ingress and registry
```
    minikube addons enable ingress
    minikube addons enable registry
```

# Create local docker registry
1. Select a server with good connectivity (and disk space) to where our minikube or kubernetes is running
2. (Optional - Secure registry)
    * Create self-signed certs (If you have paid certs use them instead)
    ```
        openssl req -newkey rsa:4096 -nodes -sha256 \
            -keyout registry_certs/domain.key -x509 -days 356 \
            -out registry_certs/domain.cert
    ```
    * start docker with certs 
    ```
        docker run -d -p 5000:5000 \
                -v $(pwd)/registry_certs:/certs \
                -v /home/dockerRegistry:/var/lib/registry \
                -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.cert \
                -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
                --restart=always --name registry registry:2
    ``` 
3. Install docker on the server. Start registry in insecure mode.
    ``` 
        docker run -d -p 5000:5000 --name docker-registry --restart=always -v /home/dockerRegistry:/var/lib/registry registry:2
    ```
The volume share ensures that even if server restarts, the images which are already pushed, are safe on the file system of the host machine.

4. (Insecure Registry) If the registry server is on a secure LAN and you dont want to configure certificates specifically, follow this :
    * On all hosts where kubernetes or minikube is running
    * add these lines to the file : /etc/docker/daemon.json
    ```
        { 
        "insecure-registries":["<REGISTRY HOST IP>:5000"] 
        }
    ``` 

5. (Optional - UI) You can also add a UI for the registry
    ```
        docker run \
            -d \
            -e ENV_DOCKER_REGISTRY_HOST=<REGISTRY HOST IP> \
            -e ENV_DOCKER_REGISTRY_PORT=5000 \
            -e ENV_DOCKER_REGISTRY_USE_SSL=1 \
            -p 0.0.0.0:80:80 \
            konradkleine/docker-registry-frontend:v2
    ```

## Build and Push images to registry
1. Build commadn - 
    ``` 
        docker build -t <iamge-name>:<tag> -f <docker file location> . 
    ```
2. Tag the image 
    ```
    docker tag <image-name>:<tag> <private-registry-image>:<tag>
    ```
3. push the image to registry. (You must be logged in, if necessary)
    ```
    docker push <private-registry-image>:<tag>
    ```

### Kubernetes/Minikube - pull image from private registry

    ``` 
        kubectl create secret docker-registry regcred --docker-server=<REGISTRY-HOST-IP> --docker-username=<docker-username> --docker-password=<docker-password>
    ```

