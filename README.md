# Setup Alfresco Content Services (ACS) v6.2 on AWS EKS

## Overview
Alfresco Content Services (ACS) is the Enterprise Content Management module of Alfresco Digital Business Platform, which is running on a set of containerized applications on Kubernetes. This document explains how to setup a Kubernetes cluster on AWS's managed Kubernetes platform: EKS, with step by step guides to setup ACS 6.2 on EKS cluster, using tool like eksctl, kubectl and helm.

## Reference Link
This setup guide is based on the following sites:
* https://docs.alfresco.com/6.1/tasks/deploy-helm-aws.html
* https://github.com/Alfresco/acs-deployment/blob/master/docs/helm-deployment-aws_eks.md
* https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

## Prerequisites
1. The setup is based on AWS's Hong Kong region.
2. Please request AWS account user's access key ID and secret from account admin, they are required for authentication in AWS command line tool (CLI), admin can goto AWS Console -> IAM -> Users -> Security Credentials -> Access Key to create access key.
3. Please request Quay.io credentials by logging a ticket in Alfresco support center, which is required to pull ACS docker images from Quay.io repository.
4. Please discuss with your AWS account's admin to decide and setup the CIDR range of the VPC and subnets that the EKS cluster will sit, it is suggested to create one subnet per each AZ in the region.
5. Ensure "DNS resolution"  and "DNS hostnames" are enabled in the VPC, and the subnets' route table has a 0.0.0.0/0 route to internet gateway, i,e, they are public subnets.
6. Create IAM role "eks-control-role", which will be assumed by the EKS cluster:
   - Goto IAM in AWS console, select service: EKS and click “EKS” in use case and click “Next”: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks-control-role.png)
   - Keep click “next” and at the final step input the role name: “eks-control-role” and click “Create Role”: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks-control-role-2.png)
7. Create IAM role “NodeInstanceRole”, which will be assumed by the EKS worker nodes:
   - Goto IAM in AWS console, select service: EC2 and click “Next”: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/nodeinstancerole.png)
   - Attach the following polices to the role:
     - AmazonEKSWorkerNodePolicy
     - AmazonEKS_CNI_Policy
     - AmazonEC2ContainerRegistryReadOnly
     - AmazonS3FullAccess
   - At the final step give the role a name “NodeInstanceRole” and create role: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/nodeinstancerole2.png)

8. Create EC2 key pair for the EKS worker node:
   - Goto EC2 in AWS console, click “key pair” at the left menu: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/ec2keypair.png)
   - Input the name and create key pair: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/ec2keypair2.png)

## Amazon EKS Cluster Setup
1. Goto “EKS” in AWS console, ensure selected region is Hong Kong: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks1.png)
2. Input the cluster name, Kubernetes version and service role:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks2.png)
3. Select the VPC and subnets for the EKS cluster:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks3.png)
4. Set the cluster security group, pick the default is fine:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks4.png)
5. Allow private access to API server endpoint only:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks5.png)
6. Set the below logging options and create the cluster:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks6.png)
7. Wait for successful creation of cluster, which means cluster status = “active”:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks7.png)
8. Scroll down to “Node Groups” and click “Add node group”:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks8.png)
9. Set node group configuration, ensure all subnets are selected:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks9.png)
10. Input SSH key pair for EKS worker nodes and allow only the EKS cluster's security group for remote access:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks10.png)
11. Set worker node compute configuration:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks11.png)
12.	Set scaling configuration:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks12.png)
13. Final step is review the group configuration and create, wait for the node group status become “active”:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks13.png)
14. Goto EC2 dashboard, there should be 3 EC2 instances running:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks14.png)

## Bastion Host Setup
A Bastion host (an EC2 instance) is setup in a public subnet of the EKS cluster's VPC, so that user from external networks can SSH to it and manage the EKS cluster using command line tool like eksctl, kubectl and helm.

1. Goto EC2 dashboard, click “launch instance”
2. Search “ami-03a9535798e343119”, this is a Linux AMI for a Bastion Host in HK region:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion2.png)
3. Select instance type, t3.micro should be fine:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion3.png)
4. Ensure a public subnet is selected:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion4.png)
5. Add 30GB storage:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion5.png)
6. Set security group, to allow SSH inbound traffic from internet:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion6.png)
7. Create a key-pair and launch the instance:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion7.png)
8. Check if the instance start successfully:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion8.png)
9. Click actions -> networking -> change security groups, add the EKS cluster's security group.
10. Click the instance and click connect, follow the instruction to SSH to the Bastion host.
11. If fail to connect pls check the subnet’s route table if there is a route to internet gateway, also check if the instance has a public DNS name and IP.
12. SSH to the instance, install the PostgesSQL client by
```bash
yum install postgresql-devel
```

## Setup AWS CLI, eksctl, kubectl and helm
1. The following steps are done in the Bastion host.
2. Run the below command to install / upgrade AWS Command Line Tool (CLI)
```bash
pip install awscli –upgrade –user
```
3. Setup AWS account credential for CLI by “AWS Configure”:
```
AWS Access Key ID [None]: <Your Access Key ID>
AWS Secret Access Key [None]: <Your Access Key Secret>
Default region name [None]: ap-east-1
Default output format [None]: json
```
4. Download and extract the latest release of eksctl with the following command:
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
5. Move the extracted binary to /usr/local/bin:
```bash
sudo mv /tmp/eksctl /usr/local/bin
```
6. Test that your installation was successful with the following command:
```bash
eksctl version
```
7. Follow the below guide to setup kubectl, for Linux and Kubernetes 1.14: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
8. To setup authentication between kubectl and the EKS cluster, use the AWS CLI update-kubeconfig command to create kubeconfig for your cluster, please replace **acs** with your EKS cluster name:
```bash
aws eks –region ap-east-1 update-kubeconfig –name acs
```
9. Test your configuration:
```bash
kubectl get svc
```
![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/kubectl9.png)

## Amazon Elastic File System (EFS) Setup
EFS is mainly used to store the full text indexes of documents stored in Alfresco content services (the Solr Indexes)
1. Goto EFS in AWS console, select the VPC of EKS cluster and click all AZs: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/efs1.png)
2. Follow the setup process and create the EFS:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/efs2.png)
3. After EFS is created please drop down the EFS DNS name:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/efs3.png)
4. Click Actions -> Manage network access, add the EKS cluster security group to all mount targets: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/efs4.png)

## Amazon Aurora (PostgreSQL) Setup
The Aurora database is used to store meta-data of the documents in Alfresco content services.
1. Goto RDS in AWS console, click “create database” and choose engine type = “Aurora”:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/rds2.png)
2. Choose Amazon Aurora with PostgreSQL compatibility:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/rds3.png)
3. Input DB cluster name, db username and password:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/rds4.png)
4. Choose the instance type:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/rds5.png)
5. No need to create Aurora Replica:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/rds6.png)
6. Select the VPC of EKS cluster:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/rds7.png)
7. Ensure security group of EKS cluster is added and create the database: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/rds7b.png)
8. Wait for Aurora database cluster to create successfully:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/rds8.png)
9. Click the database cluster name and you can see the "write" database endpoint at the bottom: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/rds9.png)
10. Login to the Bastion host and execute the following command to create a default database:
```
createdb -h <Aurora db cluster endpoint> -p 5432 -U <db username> <db username>
```
11. Connect to the Aurora's Postgres database by:
```
psql -h <Aurora db cluster endpoint> -p 5432 -U <db username> <db username>
```
12. If login successful, that means database setup successfully! and can quit the console by:
```
\q
```

## AWS Certificate Manager Setup
Data in transit between end user and alfresco digital platform is protected by HTTPS, therefore a SSL/TLS certificate for the application domain is required.
1. Go to ACM in AWS console, choose provision a SSL/TLS certificate, and request a public certificate or import an existing one.
2. After the certificate is issued/imported, drop down the ARN:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/acm3.png)
 
## Alfresco Content Services Setup
1. Run the following command to install helm, a package manager on Kubernetes:
```bash
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > install-helm.sh
chmod u+x install-helm.sh
./install-helm.sh
```
2. Use a kubectl command with an external YAML file to create role-based access control (RBAC) configuration for Tiller:
   - First, create the external YAML file:
   ```bash
   cat <<EOF > tiller-rbac-config.yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: tiller
     namespace: kube-system
   ---
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRoleBinding
   metadata:
     name: tiller
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
     - kind: ServiceAccount
       name: tiller
       namespace: kube-system
   EOF
   ```
   - Next, apply the RBAC configuration for Tiller via a kubectl command:
   ```bash
   kubectl create -f tiller-rbac-config.yaml
   ```
   Finally initialize tiller:
   ```bash
   helm init --service-account tiller
   ```
3. Set the namespace variable for Alfresco content services, replace **acs** below with your prefered name:
```bash
export DESIREDNAMESPACE=acs
```
4. Create the namespace in the EKS cluster:
```bash
kubectl create namespace $DESIREDNAMESPACE
```
5. Create the nginx ingress controller and AWS classic load balancer by:
   - Replace **your-ssl-cert-arn** below with the ARN of the certificate from AWS Certificate Manager:
   ```bash
   export AWS_CERT_ARN="your-ssl-cert-arn"
   ```
   - Replace **myacs.example.com** below with the hostname of Alfresco content services:
   ```bash
   export AWS_EXT_URL="myacs.example.com"
   ```
   - Install the ingress controller by the below helm command:
   ```bash
   helm install stable/nginx-ingress \
   --version 0.14.0 \
   --set controller.scope.enabled=true \
   --set controller.scope.namespace=$DESIREDNAMESPACE \
   --set rbac.create=true \
   --set controller.config."force-ssl-redirect"=\"true\" \
   --set controller.config."server-tokens"=\"false\" \
   --set controller.service.targetPorts.https=80 \
   --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-backend-protocol"="http" \
   --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-ssl-ports"="https" \
   --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-ssl-cert"=$AWS_CERT_ARN \
   --set controller.service.annotations."external-dns\.alpha\.kubernetes\.io/hostname"=$AWS_EXT_URL \
   --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-ssl-negotiation-policy"="ELBSecurityPolicy-TLS-1-2-2017-01" \
   --set controller.publishService.enabled=true \
   --namespace $DESIREDNAMESPACE
   ```
   - Check nginx-ingress installed successfully by:
   ```
   kubectl get pods --namespace acs
   ```
   Keep checking until the two pods changed to `running` status
   - Goto EC2 dashboard in AWS console, click load balancer, you will see a load balancer is setup, drop down the DNS name of it.

6. Setup EFS with EKS cluster, please replace **efs-dns-name** with your EFS's DNS name:
```bash
export EFS_SERVER="efs-dns-name"

helm install stable/nfs-client-provisioner \
--name alfresco-nfs-provisioner \
--set nfs.server="$EFS_SERVER" \
--set nfs.path="/" \
--set storageClass.name="nfs-client" \
--set storageClass.archiveOnDelete=false
```
7.	Install docker by:
```bash
sudo yum install docker
```
8.	Login to Quay.io and generate a base64 value for your dockercfg using one of the following methods, this will allow Kubernetes to access Quay.io:
```bash
docker login quay.io
cat ~/.docker/config.json | base64
```
9.	Create a file secrets.yaml with the following command, remember to replace **your-base64-string** with the string that generated in last step:
```bash
cat <<EOF > secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: quay-registry-secret
type: kubernetes.io/dockerconfigjson
data:
# Docker registries config json in base64 - to do this, run:
# cat ~/.docker/config.json | base64
  .dockerconfigjson: your-base64-string
EOF
```
10.	Deploy the secret into the Kubernetes cluster:
```bash
kubectl create -f secrets.yaml --namespace $DESIREDNAMESPACE
```
11. You should see the following output:
```
secret "quay-registry-secret" created
```
12. Add the Alfresco Helm charts repository:
```bash
helm repo add alfresco-incubator https://kubernetes-charts.alfresco.com/incubator
helm repo add alfresco-stable https://kubernetes-charts.alfresco.com/stable
```
13. Deploy Alfresco Content Services using the following set of commands:
```bash
# Replace acs with namespace of ACS
export DESIREDNAMESPACE=acs

# DNS name of the ACS cluster or the AWS ELB DNS name if you do not intend to create one.
# Replace myacs.example.com with your hostname of alfresco content services
export EXTERNALHOST="myacs.example.com"

# Replace efs-dns-name with EFS DNS Name
export EFS_SERVER="efs-dns-name"

# Alfresco Admin password is encoded in MD4, set the default admin password to "alfrescopoc!"
export ALF_ADMIN_PWD="301aecafe3dc996e59de880cc5a8a8d4"

# Alfresco RDS database endpoint
export ALF_DB_URL="alfresco.cluster-abcdefg.ap-east-1.rds.amazonaws.com:5432"
# Alfresco RDS database username
export ALF_DB_USER="alfresco"
# Alfresco RDS database password
export ALF_DB_PWD="alfrescopoc"

# Install ACS on EKS cluster with S3 connector
helm install alfresco-incubator/alfresco-content-services \
--set externalProtocol="https" \
--set externalHost="$EXTERNALHOST" \
--set externalPort="443" \
--set repository.adminPassword="$ALF_ADMIN_PWD" \
--set alfresco-infrastructure.persistence.storageClass.enabled=true \
--set alfresco-infrastructure.persistence.storageClass.name="nfs-client" \
--set alfresco-infrastructure.alfresco-infrastructure.nginx-ingress.enabled=false \
--set alfresco-search.resources.requests.memory="2500Mi",alfresco-search.resources.limits.memory="2500Mi" \
--set alfresco-search.environment.SOLR_JAVA_MEM="-Xms2000M -Xmx2000M" \
--set postgresql.postgresPassword="$ALF_DB_PWD" \
--set persistence.solr.data.subPath="$DESIREDNAMESPACE/alfresco-content-services/solr-data" \
--set persistence.repository.enabled=false \
--set s3connector.enabled=true \
--set s3connector.config.bucketName=hkt-alfresco-poc \
--set s3connector.secrets.encryption=kms \
--set global.alfrescoRegistryPullSecrets=quay-registry-secret \
--namespace=$DESIREDNAMESPACE
```
14. You will see the following output at the end:
```
You can access all components of Alfresco Content Services Community using the same root address, but different paths as follows:

  Content: https://myacs.exmaple.com:443/alfresco
  Share: https://myacs.exmaple.com:443/share
  Api-Explorer: https://myacs.exmaple.com:443/api-explorer
  Sync service: https://myacs.exmaple.com:443/syncservice/healthcheck 
```
15. Check the installation status by following command, replace **acs** with your EKS cluster name, Keep checking until all pods' status changed to `Running`
```
kubectl get pods --namespace acs
```
16. Go to the external host site, for example: https://myacs.myexample.com/share, the Alfresco content services site should be up and running.



