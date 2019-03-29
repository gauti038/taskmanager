CircleCI Build Status : [![CircleCI](https://circleci.com/gh/gauti038/taskmanager/tree/master.svg?style=svg)](https://circleci.com/gh/gauti038/taskmanager/tree/master)

# DOCKER CHALLENGE

## Assumptions
1. Run the application on non Kubernetes environment - (localhost or ubuntu server)
2. Java 7+, Maven 3.5+ and docker-compose (latest) are already installed on the machine
3. Only web app must be exposed. DB and scheduler must not be accessible on localhost.
4. Containers must talk to each other using only service names
5. Size of final docker images must be as small as possible (around 200 MB in this case)

## Option 1 - Java, Maven and docker-compose 

## Method and Solution
1. clone this repo and do a maven build
    ``` 
        git clone git@github.com:gauti038/taskmanager.git 
        cd taskmanager 
        mvn clean install -Pweb 
        mvn clean install -Pscheduler
    ```
2. These commands will download and build all the required files
3. Start docker compose 
    ```
        docker-compose up --build
    ```
4. The docker compose uses docker files - Dockerfile-web-localhost & Dockerfile-scheduler-localhost
5. The dockerfiles just copies the required jar files into a alpine base image (reduce image size)
6. Open http://localhost:8080 for the web application 
7. Open [Weavescope](http://localhost:4040/#!/state/{%22pinnedMetricType%22:%22Memory%22,%22topologyId%22:%22containers%22}) for weavescope (an advanced realtime container interations UI)
8. dependencies are set to each of the containers

## Option 2 - Only docker-compose

## Method and Solution
1. clone this repo and do a maven build
    ``` 
        git clone git@github.com:gauti038/taskmanager.git 
        cd taskmanager 
        # comment proper sections of docker-compose.yml file
        docker-compose up --build
    ```
2. These commands will start multi-stage docker builds to reduce image sizes
3. The docker compose uses docker files - Dockerfile-web-compose & Dockerfile-scheduler-compose
4. The dockerfiles first use maven image (large in size) and does a maven build. The jar is then copied onto a light-weight java alpine docker image 
5. The disadvantage of this method is - it takes too much time to build (15+ mins) 
6. Once the containers start, open http://localhost:8080 for the web application 
7. Open [Weavescope](http://localhost:4040/#!/state/{%22pinnedMetricType%22:%22Memory%22,%22topologyId%22:%22containers%22}) for weavescope (an advanced realtime container interations UI)
8. dependencies are set to each of the containers

Weave scope is a beautiful UI which shows live interactions between pods. 
more at https://www.weave.works/oss/scope/ 

# Minikube Solution

1. Make sure minikube is installed with required addons - 
    [Minikube Installation Steps](k8s/README.md)

2. Use minikube registry. 
    ``` 
        eval $(minikube docker-env) 
    ```
    This sets a list of Bash environment variable exports to configure your local environment to re-use the Docker daemon inside the Minikube instance. (Basically sharing the registry)

3. Maven build and install
    ```
    mvn clean install -Pweb
    mvn clean install -Pscheduler
    ```
4. Build the required docker images
    ```
    docker build -t web:minikube -f ./Dockerfile-web-localhost .
    docker build -t scheduler:minikube -f ./Dockerfile-scheduler-localhost . 
    ```
    This builds the image and stores them in the docker registry of minikube.

5. Pull postgres image so that minikube has all images when we start deploying containers
    ``` 
        docker pull postgres 
    ```

6. Apply the Kubernetes yaml files. I have only used simple YAML files for deployment and not used any helm charts.         All the deployments have readiness probe and liveness Probe.
    * Postgres - has a service. shares a volume for data incase of restarts. Readiness probe is a psql command. 
    * Web - has a service which is exposed on ingress. Readiness probe is tcp socket on 8080. Has resource limits for autoscaling. Autoscaler based on CPU at 70%.
    * Scheduler - doesnt need a service, since not called or exposed. Readiness probe is tcp socket on 8080. Can be autoscaled based on resources. Autoscaler based on CPU at 70%.
    ```
    kubectl create -f k8s/
    ```
    Wait for some time till all pods start running. (Dont worry if some of them restart)

7. Add domain name needed for the ingress IP on your localhost
    ``` 
        echo "$(minikube ip) omnius-challenge.demo" | sudo tee -a /etc/hosts 
    ```
8. Open http://omnius-challenge.demo/ on browser and you can access webapp 

9. The same yamls can be used on Kubernetes with appropriate changes to docker registry.

10. Commands to manually scale a deployment 
    ```
        kubectl scale deployment <deployment name> --replicas=<scale-count>

        Eg: kubectl scale deployment web --replicas=3
    ```
11. Command to update the image for a deployment.
    ```
        kubectl set image deployment <deployment name> <container name>=<image name>:<tag>
        
        Eg: kubectl set image deployment web web=web:minikube2
    ```
12. Auto Complete Kubectl commands on bash 
    ```
        echo 'source <(kubectl completion bash)' >>~/.bashrc
        source ~/.bashrc
    ```






