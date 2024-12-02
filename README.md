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

        - Create an IAM policy (say, EksAllAccess) to provide complete access to EKS Services

        ```
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": "eks:*",
                    "Resource": "*"
                },
                {
                    "Action": [
                        "ssm:GetParameter",
                        "ssm:GetParameters"
                    ],
                    "Resource": [
                        "arn:aws:ssm:*:<account_id>:parameter/aws/*",
                        "arn:aws:ssm:*::parameter/aws/*"
                    ],
                    "Effect": "Allow"
                },
                {
                    "Action": [
                    "kms:CreateGrant",
                    "kms:DescribeKey"
                    ],
                    "Resource": "*",
                    "Effect": "Allow"
                },
                {
                    "Action": [
                    "logs:PutRetentionPolicy"
                    ],
                    "Resource": "*",
                    "Effect": "Allow"
                }        
            ]
        }
        ```

        - Create an IAM policy (say, IamLimitedAccess) to provided limited access to AWS IAM

        ```
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:CreateInstanceProfile",
                        "iam:DeleteInstanceProfile",
                        "iam:GetInstanceProfile",
                        "iam:RemoveRoleFromInstanceProfile",
                        "iam:GetRole",
                        "iam:CreateRole",
                        "iam:DeleteRole",
                        "iam:AttachRolePolicy",
                        "iam:PutRolePolicy",
                        "iam:UpdateAssumeRolePolicy",
                        "iam:AddRoleToInstanceProfile",
                        "iam:ListInstanceProfilesForRole",
                        "iam:PassRole",
                        "iam:DetachRolePolicy",
                        "iam:DeleteRolePolicy",
                        "iam:GetRolePolicy",
                        "iam:GetOpenIDConnectProvider",
                        "iam:CreateOpenIDConnectProvider",
                        "iam:DeleteOpenIDConnectProvider",
                        "iam:TagOpenIDConnectProvider",
                        "iam:ListAttachedRolePolicies",
                        "iam:TagRole",
                        "iam:UntagRole",
                        "iam:GetPolicy",
                        "iam:CreatePolicy",
                        "iam:DeletePolicy",
                        "iam:ListPolicyVersions"
                    ],
                    "Resource": [
                        "arn:aws:iam::<account_id>:instance-profile/eksctl-*",
                        "arn:aws:iam::<account_id>:role/eksctl-*",
                        "arn:aws:iam::<account_id>:policy/eksctl-*",
                        "arn:aws:iam::<account_id>:oidc-provider/*",
                        "arn:aws:iam::<account_id>:role/aws-service-role/eks-nodegroup.amazonaws.com/AWSServiceRoleForAmazonEKSNodegroup",
                        "arn:aws:iam::<account_id>:role/eksctl-managed-*"
                    ]
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetRole",
                        "iam:GetUser"
                    ],
                    "Resource": [
                        "arn:aws:iam::<account_id>:role/*",
                        "arn:aws:iam::<account_id>:user/*"
                    ]
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:CreateServiceLinkedRole"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "iam:AWSServiceName": [
                                "eks.amazonaws.com",
                                "eks-nodegroup.amazonaws.com",
                                "eks-fargate.amazonaws.com"
                            ]
                        }
                    }
                }
            ]
        }
        ```

       
        - Create an IAM role (say, EKSClientRole) with the following policies, and attach it to the EC2 instance, serving as the EKS client 

            - IamLimitedAccess
            - EksAllAccess
            - AWSCloudFormationFullAccess (AWS Managed Policy)
            - AmazonEC2FullAccess (AWS Managed Policy)

        - Create a security group for the EC2 instance allowing inbound SSH traffic

        - Launch the EC2 Instance (EKS Client), with the following configs

        ```
        aws ec2 run-instances --image-id ami-0166fe664262f664c --instance-type t2.medium --key-name SSH1 --security-group-ids sg-013069d9f0783aca5 --iam-instance-profile Name=EKSClientRole --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=EKSClient}]" --count 1 --region us-east-1 --user-data  file://./phase1/eksClientStartupScript.sh
        ```   

        - SSH into the EC2 instance 

        - Create the EKS Cluster using the following command 

        ```
        eksctl create cluster -f  EKS-and-Monitoring-with-OpenTelemetry/phase1/eks-cluster-deployment.yaml
        ```
        
        - Deploy the application to the EKS Cluster

        ```
        kubectl apply -f EKS-and-Monitoring-with-OpenTelemetry/phase1/opentelemetry-demo.yaml
        ```

- Validate the deployment:

    - Ensure all pods and services are running as expected in the otel-demo namespace.

        ```
        kubectl get all -n otel-demo
        ```

    - Access application endpoints through port-forwarding or service

        - Accessing Grafana

            - Forward a local port on the EKS Client to a port on a service running within a Kubernetes cluster

            ```
            kubectl port-forward svc/opentelemetry-demo-grafana 8080:80 -n otel-demo

            ```

            - Sets up a local port forwarding from your local machine to the remote EC2 instance (EKS client), to securely access a service running on that EC2 instance.

            ```
            ssh -i "SSH1.pem" -L 8080:127.0.0.1:8080 ec2-user@ec2-3-86-28-24.compute-1.amazonaws.com
            ```

            - Access the application from the local machine 
            ```
            http://localhost:8080       
            ```

    - Collect the cluster details, including node and pod

        ```
        kubectl get nodes -o wide
        kubectl describe nodes
        kubectl get pods -n otel-demo
        ```

    - Do not delete the EKS cluster unless explicitly

#### Deliverables
 
- Screenshot of the EKS cluster configuration details (number of nodes, instance type, ).

    ![Cluster Config](/screenshots/phase1/cluster-config.png)

    ![Cluster Node Group Config](/screenshots/phase1/cluster-node-group-config.png)

- Screenshot of the EC2 instance used as the EKS client (instance type, storage, ).

    ![EKS Client](/screenshots/phase1/eks-client.png)

- Screenshot of kubectl get all -n otel-demo showing the status of pods, services, and deployments.

    ![Kubectl Get All](/screenshots/phase1/kubectl-get-all.png)

- Screenshot of logs from key application pods to confirm successful
    
    ```
    kubectl logs <pod-name> -n otel-demo
    ```

    - To Retrieve the last 75 log messages from the opentelemetry-demo-grafana-69b6bd5dd4-bvs5k pod

    ```
    kubectl logs opentelemetry-demo-grafana-69b6bd5dd4-bvs5k -n otel-demo --tail=75
    ``` 

    ![Grafana Pod Log](/screenshots/phase1/grafana-pod-log.png)

    - Similarly, retrieving the last 75 log messages from the opentelemetry-demo-otelcol-5c757cfcf-4vkfp and opentelemetry-demo-prometheus-server-57cd8f9d46-qnd27 pods

    ```
    kubectl logs opentelemetry-demo-otelcol-5c757cfcf-4vkfp -n otel-demo --tail=75
    kubectl logs opentelemetry-demo-prometheus-server-57cd8f9d46-qnd27 -n otel-demo --tail=75
    ``` 

    ![Prometheus Server Pod Log](/screenshots/phase1/prometheus-server-pod.png)

- Exported Kubernetes manifest (opentelemetry-demo.yaml).

- Screenshot of accessible application follow the project documentation link.

    - Accessing Grafana 

        - Port forwarding on the EKS Client to the service running Kubernetes

        ![Kubectl port forwarding - Grafana](/screenshots/phase1/kubectl-port-forwarding-grafana.png)

        - Port forwarding from Local Machine to remote EC2 instance (EKS Client)

        ![Local port forwarding - Grafana](/screenshots/phase1/local-port-forwarding-grafana.png)

        - Accessing the application Locally 

        




 


