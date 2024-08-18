# Kubernetes and Docker project 

The below set up serves a purpose to respond to a client request. If the client sends out a **hello** request , in return the server responds with the **hi** response.
To achieve this setup we will work with linux machine with required dependencies for Docker and kubernetes. 

Docker is an open-source containerization platform by which you can package your application.

Kubernetes is a platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. 

We will begin with creating a flask application packagind and pushing it to the docker repository and running a kubernetes deployment based on the docker image. To balance the incomming traffic to our application we will use the round robin algorithm which work on the principle of first come first serve basis and help in balancing  out the network traffic to our application.

## Prerequisites

1. A linux Machine 

## Setting up Linux Machine

1. Create a linux virtual machine with the required ports opened up.


## Installing Docker

1. Install the docker in Ubuntu Linux using the following commands:


        sudo apt-get install -y docker.io

        

# Install the kubernetes(kubeadm,kubelet,kubectl)

1. Install the kubectl in ubuntu machine 
    
    1.1 Download the Kubectl checksum file:


         curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

    1.2 Validate the Kubectl


        echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
 
    1.3 Install the Kubectl

         sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

    1.4 Test that Kubectl is installed correctly and also check it have latest version


        kubectl version --client
   
2. These instructions are for Kubernetes v1.30.


refrence taken from https://v1-30.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/


3. Initialize Kubernetes Master Node:

    3.1 Initialize the master node:

       sudo kubeadm init

    3.2  Set up kubectl:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config


4. Install Networking Plugin:

    4.1 Install a networking plugin such as Weave or Flannel:

        kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml



##  Deploy Applications (Servers) in Kubernetes

1. Create Docker Images for Servers:

    1.1 Create a Directory for Your Docker Project

    1.1.1  Create a project directory:
        
        mkdir my-docker-project
        cd my-docker-project

    1.1.2 Create a subdirectory for each server:

        mkdir server1 server2

    1.2 Create the Server Applications

    need to create simple Python Flask applications for Server 1 and Server 2.

    1.2.1 Create the Server 1 Python file:

       nano server1/server1.py

    after that paste this code in the file:


          from flask import Flask

            app = Flask(__name__)

            @app.route('/', methods=['GET'])
            def hello():
                return "Hi from Server 1!"

            if __name__ == "__main__":
                app.run(host='0.0.0.0', port=5000)


    1.2.2  Create the Server 2 Python file:
       
       nano server2/server2.py

    after that paste the code 


        from flask import Flask

            app = Flask(__name__)

            @app.route('/', methods=['GET'])
            def hello():
                return "Hi from Server 2!"

            if __name__ == "__main__":
                app.run(host='0.0.0.0', port=5000)

    1.3 Create Dockerfiles for Each Server

    create a Dockerfile in each server's directory.

    1.3.1 Create the Dockerfile for Server 1:

        nano server1/Dockerfile

    and now paste the code 


        FROM python:3.8-slim

        WORKDIR /app

        COPY server1.py /app/

        RUN pip install flask

        EXPOSE 5000

        CMD ["python", "server1.py"]


    1.3.2  Create the Dockerfile for Server 2:
       
       nano server2/Dockerfile


    after that paste that code 

       FROM python:3.8-slim

        WORKDIR /app

        COPY server2.py /app/

        RUN pip install flask

        EXPOSE 5000

        CMD ["python", "server2.py"]



    1.4  Build the Docker Images
      
    1.4.1 Navigate to the server1 directory and build the image:


            cd server1
            docker build -t server1-image .

    
    1.4.2 Navigate to the server2 directory and build the image:
        

        cd ../server2
        docker build -t server2-image .

    1.5     Run the Docker Containers

    1.5.1   Run the Server 1 container:

        docker run -d -p 5001:5000 --name server1-container server1-image


    1.5.2 Run the Server 2 container:

        docker run -d -p 5002:5000 --name server2-container server2-image

    1.6 Verify the Running Containers

    1.6.1 List running containers:

        docker ps

    1.6.2 Test the applications by visiting the following URLs in your browser:

        Server 1: http://localhost:5001
        Server 2: http://localhost:5002

    
    1.7 Push Docker Images to a Registry

    1.7.1 Tag the images:

        docker tag server1-image anshulsharma05/server1-image:latest
        docker tag server2-image anshulsharma05/server2-image:latest

    1.7.2 Login to Docker and enter username and password

        docker login

    1.7.3 Push the images:

        docker push anshulsharma05/server1-image:latest
        docker push anshulsharma05/server2-image:latest


2. Create Kubernetes Deployments

   2.1 for Server 1:

        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: server1-deployment
        spec:
        replicas: 1
        selector:
            matchLabels:
            app: server1
        template:
            metadata:
            labels:
                app: server1
            spec:
            containers:
            - name: server1-container
                image: anshulsharma05/server1-image 
                ports:
                - containerPort: 80

    2.2 for Server 2:

        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: server2-deployment
        spec:
        replicas: 1
        selector:
            matchLabels:
            app: server2
        template:
            metadata:
            labels:
                app: server2
            spec:
            containers:
            - name: server2-container
                image: anshulsharma05/server2-image
                ports:
                - containerPort: 80


    2.3 deploy the servers to the kubernetes cluster:

        kubectl apply -f server1-deployment.yaml
        kubectl apply -f server2-deployment.yaml


3. Set Up a Load Balancer

    3.1 Use NGINX as a Load Balancer having approach of Round Robin:

    Deploy NGINX as a Kubernetes Deployment to serve as the load balancer.

        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: nginx-loadbalancer
        spec:
        replicas: 1
        selector:
            matchLabels:
            app: nginx-loadbalancer
        template:
            metadata:
            labels:
                app: nginx-loadbalancer
            spec:
            containers:
            - name: nginx
                image: nginx:latest
                ports:
                - containerPort: 80
                volumeMounts:
                - name: nginx-config-volume
                mountPath: /etc/nginx/nginx.conf
                subPath: nginx.conf
        volumes:
        - name: nginx-config-volume
            configMap:
            name: nginx-config


    Configure the NGINX ConfigMap to balance requests between Server 1 and Server 2 based on their current load.

        
        apiVersion: v1
        kind: ConfigMap
        metadata:
        name: nginx-config
        data:
        nginx.conf: |
            upstream backend {
                server server1-service:80;
                server server2-service:80;
            }
            server {
                listen 80;
                location / {
                    proxy_pass http://backend;
                }
            }


    Apply the ConfigMap and Deployment:

        kubectl apply -f nginx-config.yaml
        kubectl apply -f nginx-loadbalancer.yaml


4. Expose the Services

    4.1 Create Services for Servers:

    Create Kubernetes Services to expose the two servers.

    for Server 1:

        apiVersion: v1
        kind: Service
        metadata:
        name: server1-service
        spec:
        selector:
            app: server1
        ports:
            - protocol: TCP
            port: 80
            targetPort: 80

    for Server 2:

        apiVersion: v1
        kind: Service
        metadata:
        name: server2-service
        spec:
        selector:
            app: server2
        ports:
            - protocol: TCP
            port: 80
            targetPort: 80

    Deploy the Services:

        kubectl apply -f server1-service.yaml
        kubectl apply -f server2-service.yaml

    4.2  Expose NGINX Load Balancer:

    Expose the NGINX load balancer using a Kubernetes Service:

        apiVersion: v1
        kind: Service
        metadata:
        name: nginx-service
        spec:
        selector:
            app: nginx-loadbalancer
        ports:
            - protocol: TCP
            port: 80
            targetPort: 80

    Deploy the NGINX Service:

        kubectl apply -f nginx-service.yaml

5. Implement Load-Based Distribution (Dynamic Load Balancing)

    5.1 Set Up Monitoring with Prometheus and Grafana
     
    Deploy Prometheus using Helm:

        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
        helm repo update
        helm install prometheus prometheus-community/prometheus

    Deploy Grafana using helm:
       
        helm repo add grafana https://grafana.github.io/helm-charts
        helm repo update
        helm install grafana grafana/grafana

    5.2 Configure Prometheus to Monitor Servers:

    Edit the Prometheus configuration to scrape metrics from Server 1 and Server 2.

        scrape_configs:
            - job_name: 'server1'
                static_configs:
                - targets: ['server1-service:8090']

            - job_name: 'server2'
                static_configs:
                - targets: ['server2-service:8090']


    now monitor server load 

    Set up Prometheus to monitor CPU and memory usage of Server 1 and Server 2.
    Use Grafana to visualize these metrics.

    5.3 Adjust NGINX Load Balancing Based on Load:

        # Fetch load metrics from Prometheus
        SERVER1_LOAD=$(curl -s 'http://prometheus-server/api/v1/query?query=avg_over_time(node_cpu[5m])')
        SERVER2_LOAD=$(curl -s 'http://prometheus-server/api/v1/query?query=avg_over_time(node_cpu[5m])')

        # Adjust weights based on load
        if [ $SERVER1_LOAD -lt $SERVER2_LOAD ]; then
            WEIGHT1=2
            WEIGHT2=1
        else
            WEIGHT1=1
            WEIGHT2=2
        fi

        # Update NGINX ConfigMap
        kubectl patch configmap nginx-config --patch \
        "data:
        nginx.conf: |
            events {}
            http {
                upstream backend {
                    server server1-service:8090 weight=$WEIGHT1;
                    server server2-service:8090 weight=$WEIGHT2;
                }
                server {
                    listen 80;
                    location / {
                        proxy_pass http://backend;
                    }
                }
            }"

        # Reload NGINX to apply changes
        kubectl rollout restart deployment/nginx


6. test the set up 

    6.1 Generate Traffic:
    to generate traffic use tool like 'curl','apache Bench' to send http request

    here is the script for that

 
       curl http://<nginx-service-ip>:8080/
 
    6.2 After this Monitor Load Distribution:

    use can see grafana to see how load is distributed accross servers 
