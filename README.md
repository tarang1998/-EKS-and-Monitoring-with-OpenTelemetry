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

            ![EKSClientRole](/screenshots/phase1/eksclientRole-IAMRole.png)


        - Create a security group for the EC2 instance allowing inbound SSH traffic

        - Launch the EC2 Instance (EKS Client), with the following configs

        ```
        aws ec2 run-instances --image-id ami-0166fe664262f664c --instance-type t2.medium --key-name SSH1 --security-group-ids sg-013069d9f0783aca5 --iam-instance-profile Name=EKSClientRole --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=EKSClient}]" --count 1 --region us-east-1 --user-data  file://./phase1/eksClientStartupScript.sh
        ```   

        - SSH into the EC2 instance 

        - Create the EKS Cluster using the following command 

        ```
        eksctl create cluster -f  /EKS-and-Monitoring-with-OpenTelemetry/phase1/eks-cluster-deployment.yaml
        ```

        ![EKS Cluster creation](/screenshots/phase1/eksctl-cluster-creation.png)

        - On occurrence of the warning while trying to access the cluster details from the AWS console as a root user 

            > Your current IAM principal doesn’t have access to Kubernetes objects on this cluster. This may be due to the current user or role not having Kubernetes RBAC permissions to describe cluster resources or not having an entry in the cluster’s auth config map. Learn more 

            - Check the following [article](https://stackoverflow.com/questions/70787520/your-current-user-or-role-does-not-have-access-to-kubernetes-objects-on-this-eks)

        
        - Deploy the application to the EKS Cluster

        ```
        kubectl apply --namespace otel-demo -f /EKS-and-Monitoring-with-OpenTelemetry/phase1/opentelemetry-demo.yaml
        ```

        ![Kubectl Config](/screenshots/phase1/kubectl-configuration.png)


- Validate the deployment:

    - Ensure all pods and services are running as expected in the otel-demo namespace.

        ```
        kubectl get all -n otel-demo
        ```

    - Access application endpoints through port-forwarding or service

        - Accessing the application 

            - Forward a local port on the EKS Client to a port on a service running within a Kubernetes cluster. This command allows the port-forwarding to be accessed from any IP address, not just localhost

            ```
            kubectl port-forward svc/opentelemetry-demo-frontendproxy 8080:8080 --namespace otel-demo --address 0.0.0.0
            ```

            - Access the application  
            ```
            http://<instance-public-ip>:8080    
            http://<instance-public-ip>:8080/grafana 
            http://<instance-public-ip>:8080/jaeger/ui/search
            http://<instance-public-ip>:8080/loadgen/   
            http://<instance-public-ip>:8080/feature

            ```



    - Collect the cluster details, including node and pod

        ```
        kubectl get nodes -o wide
        ```
            
        ![Kubectl get nodes](/screenshots/phase1/kubectl-get-nodes.png)


        ```
        kubectl get pods -n otel-demo
        ```

        ![Kubectl get pods](/screenshots/phase1/kubectl-get-pods-1.png)


        ```
        kubectl describe nodes
        ```


    - Do not delete the EKS cluster unless explicitly

    ```
    eksctl delete cluster -f  /EKS-and-Monitoring-with-OpenTelemetry/phase1/eks-cluster-deployment.yaml
    ```

#### Deliverables
 
- Screenshot of the EKS cluster configuration details (number of nodes, instance type, ).

    ![Cluster Config](/screenshots/phase1/cluster-config.png)

    ![Cluster Node Group Config](/screenshots/phase1/cluster-node-group-config.png)

- Screenshot of the EC2 instance used as the EKS client (instance type, storage, ).

    ![EKS Client](/screenshots/phase1/eks-client.png)

- Screenshot of kubectl get all -n otel-demo showing the status of pods, services, and deployments.

    ![Kubectl Get All Namespace otel-demo](/screenshots/phase1/kubectl-get-all-n-otel-1.png)


- Screenshot of logs from key application pods to confirm successful
    
    ```
    kubectl log <pod-name>
    kubectl logs <pod-name> -n otel-demo
    ```

    - Retrieving log messages from the frontendproxy pod

    ```
    kubectl logs <front-end-proxy pod name> --tail=50 -n otel-demo
    ```

    ![Frontend Proxy Pod Log](/screenshots/phase1/kubectl-log-frontendproxy.png)
    
    - Retrieving the last 50 log messages from the grafana pod

    ```
    kubectl logs <grafana-pod-id> -n otel-demo --tail=50
    ``` 

    ![Grafana Pod Log](/screenshots/phase1/grafana-pod-log.png)

    - Similarly Retrieving the last 50 log messages from the jaeger pod

    ![Jaeger Pod Log](/screenshots/phase1/jaeger-ui-pod-log.png)

    - Flagd pod logs

    ![FlagD pod log](/screenshots/phase1/flag-d-pod-log.png)


- Exported Kubernetes manifest (opentelemetry-demo.yaml).

    - The yaml file containing the kubernetes configurations : [opentelemetry-demo.yaml](/phase1/opentelemetry-demo.yaml)

- Screenshot of accessible application follow the project documentation link.

    - Accessing Webstore 

        - Port forwarding on the EKS Client to the service running Kubernetes

        ![Kubectl port forwarding - Frontend Proxy](/screenshots/phase1/kubectl-port-forwarding-frontend-1.png)


        - Accessing the application Locally 

        ![Webstore local access](/screenshots/phase1/webstore-local-access.png)

        ![Grafana local access](/screenshots/phase1/grafana-local-access.png)

        ![LoadGen local access](/screenshots/phase1/loadgen-local-access.png)

        ![Jaeger UI Local access](/screenshots/phase1/jaeger-ui-local-access-kubectl.png)

        ![Flagd configurator UI](/screenshots/phase1/flag-d-ui.png)

    

## Phase 2: YAML Splitting and Modular Deployment

### Objective: 

Deploy the application by creating and organizing split YAML files, applying them either individually or recursively, and validating their functionality. Splitting the YAML file is crucial because if any service or pod is down, the corresponding YAML file can be reapplied to make it functional without affecting others. This approach simplifies debugging and deployment management.

### Tasks

- Create folders for resource types to organize the split YAML files by resource type (e.g., ConfigMaps, Secrets, Deployments, Services). Ensure the folder structure is logical and reflects the Kubernetes resources being deployed.

- Apply resources either individually or recursively:
    - Individually apply each file to ensure resources deploy successfully.
    - Alternatively, apply all files recursively from the root folder containing the organized files to deploy everything.

        - SSH into the instance and move to the following path : EKS-and-Monitoring-with-OpenTelemetry/phase2/deployment

        - Deploy the namespace.yaml file 

        ```
        kubectl apply -f namespace.yaml
        ```

        - Apply all the resources recursively

        ```
        kubectl apply -f ./open-telemetry --recursive --namespace otel-demo
        ```
- Validate resource deployment by checking the status of pods, services, and Debug any issues by reviewing pod logs or describing problematic resources.

    ```
    kubectl get all -n otel-demo
    ```

- Compress the organized folder of split YAML files into a ZIP file
 
### Deliverables
- Screenshots of the created folder structure containing the split YAML

    ![Folder Structure 1 ](/screenshots/phase2/folderStructure1.png)

    ![Folder Structure 2 ](/screenshots/phase2/folderStructure2.png)

    ![Folder Structure 3 ](/screenshots/phase2/folderStructure3.png)

    - Folder Structure containing the split yaml file : [link](/phase2/treeStructure.txt)


- Screenshots showing successful deployment of each resource (individually or recursively).

    ![Namespace deployment](/screenshots/phase2/kubectl-namespace-deployment.png)

    ![Recursive deployment](/screenshots/phase2/kubectl-recursive-deployment.png)

- Screenshots showing all resources running successfully, including pods, services.

    ![Kubectl get all](/screenshots/phase2/kubectl-get-all-n-otel-demo.png)

- Logs or screenshots verifying the initialization and proper functioning of application

    ![WebStore Access](/screenshots/phase2/web-store-access.png)

    ![Grafana](/screenshots/phase2/grafana.png)

    ![LoadGen](/screenshots/phase2/loadgen.png)

    ![Jaeger UI](/screenshots/phase2/jaeger-ui.png)

    ![Flag d](/screenshots/phase2/flagd.png)

    - Frontend-proxy pod logs 

    ![Frontend-proxy pod logs](/screenshots/phase2/frontend-proxy-logs.png)

    - Grafana pod log

    ![Grafana pod logs](/screenshots/phase2/grafana-pod-log.png)


- A ZIP file containing the organized and split YAML

    - Link to zip file : [zip file](/phase2/deployment.zip)

- A short report explaining the purpose of each resource, steps followed during deployment, and resolutions to any challenges Note : Manage the namespaces properly while deploying the yaml files


 



        




 


