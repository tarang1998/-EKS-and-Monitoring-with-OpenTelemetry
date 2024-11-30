# EKS-and-Monitoring-with-OpenTelemetry

- Github Link : https://github.com/open-telemetry/opentelemetry-demoLinks
- Documentation link : https://opentelemetry.io/docs/demo/Links

 
## Phase 1: Environment and Initial Application Setup

### Docker Deployment
 
- Set up a dedicated EC2 instance (minimum large instance with 16GB storage) to install and configure Docker.
- Use the docker-compose.yml file available in the GitHub repository to bring up the service


    - User Data specification for EC2 instance to clone the Github Repo, install Docker and Docker Compose, and build the images   

        ```
        #! /bin/sh
        sudo yum update -y
        sudo yum install git -y
        sudo git clone https://github.com/open-telemetry/opentelemetry-demo.git
        sudo amazon-linux-extras install docker -y
        sudo service docker start
        sudo usermod -a -G docker ec2-user # Adds the ec2-user to the docker group to allow running Docker commands without sudo
        sudo chkconfig docker on # Configures Docker to start automatically when the system boots.
        sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
        sudo chmod +x /usr/local/bin/docker-compose
        cd opentelemetry-demo/
        docker-compose up --force-recreate --remove-orphans --detach

        ```
        
- Validate that all services defined in the docker-compose.yml are up and Ensure this by:
    - Running Docker commands like docker ps and docker-compose logs.
    - Accessing application endpoints (if applicable) and confirming service
- Once the deployment is validated, delete the EC2 instance to clean up


#### Deliverables

- Screenshot of the EC2 instance details (instance type, storage).

    ![EC2 Config](/screenshots/phase1/ec2-config.png)

    ![EC2 Security Config](/screenshots/phase1/ec2-security-config.png)

- Screenshots of the services running (docker ps).

    ![docker ps](/screenshots/phase1/docker-ps.png)

- Screenshots of Docker logs showing the application's startup

    - To view last n logs from every service : docker-compose log --tail n

    ![docker-compose log --tail 5 (1)](/screenshots/phase1/docker-compose-log-1.png)

    ![docker-compose log --tail 5 (2)](/screenshots/phase1/docker-compose-log-2.png)

    - View logs of an individual service : docker log *containerId* -n *n* (Eg. Last n logs from the email service)

    ![docker log <containerId> -n 50](/screenshots/phase1/docker-container-logs.png)


- Screenshot of accessible application

    - Web Store

    ![web store](/screenshots/phase1/webstore.png)

    - Grafana
    
    ![Grafana](/screenshots/phase1/grafana.png)

    - Load Generator UI
    
    ![Load Generator UI](/screenshots/phase1/loadgen.png)

    - Jaeger UI
    
    ![Jaeger UI](/screenshots/phase1/jaeger-ui.png)

    - Flagd configurator UI
    
    ![Flagd configurator UI](/screenshots/phase1/flagd-configurator-ui.png)

### Kubernetes Setup Tasks

- Set up an EKS Cluster in AWS with:
    - At least 2 worker
    - Instance type: large for worker nodes.
- Deploy the application to the EKS cluster using the provided opentelemetry-demo.yaml manifest file.
- Use a dedicated EC2 instance as the EKS client to:
    - Install and configure kubectl, AWS CLI, and eksctl for cluster
    - Run all Kubernetes commands from the EC2 instance (not from a local machine).

        - User Data specification for EC2 instance to install AWS CLI, kubectl and eksctl

        ```
        #!/bin/bash

        # Install kubectl
        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
        rm kubectl

        # Install eksctl
        curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_amd64.tar.gz"
        tar -xvzf /tmp/eksctl.tar.gz -C /tmp
        sudo mv /tmp/eksctl /usr/local/bin

        ```   

- Validate the deployment:
    - Ensure all pods and services are running as expected in the otel-demo namespace.
    - Access application endpoints through port-forwarding or service
    - Collect the cluster details, including node and pod
    - Do not delete the EKS cluster unless explicitly

#### Deliverables
 
- Screenshot of the EKS cluster configuration details (number of nodes, instance type, ).
- Screenshot of the EC2 instance used as the EKS client (instance type, storage, ).
- Screenshot of kubectl get all -n otel-demo showing the status of pods, services, and deployments.
- Screenshot of logs from key application pods to confirm successful
- Exported Kubernetes manifest (opentelemetry-demo.yaml), if
- Screenshot of accessible application follow the project documentation link.
 


