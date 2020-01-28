# Setup Alfresco Content Services (ACS) v6.2 on AWS EKS

## Overview
Alfresco Content Services (ACS) is the Enterprise Content Management module of Alfresco Digital Business Platform, which is running on a set of containerized applications on Kubernetes. This document explains how to setup a Kubernetes cluster on AWS's managed Kubernetes platform: EKS, with step by step guides to setup ACS on EKS cluster, using tool like eksctl, kubectl and helm.

## Prerequisites
1. The setup is based on AWS's Hong Kong region.
2. Please request AWS account user's access key ID and secret from account admin, they are required for authentication in AWS command line tool (CLI)
3. Please request Quay.io credentials by logging a ticket in Alfresco support center, which is required to pull ACS docker images from Quay.io repository.
4. Please discuss with your AWS account's admin to decide and setup the CIDR range of the VPC and subnets that the EKS cluster will sit, it is suggested to create one subnet for each AZ in the region, i.e. 3 subnets.
5. Ensure "DNS resolution"  and "DNS hostnames" are enabled in the VPC.
6. Create IAM role "eks-control-role", which will be assumed by the EKS cluster:
   - Goto IAM in AWS console, select service: EKS and click “EKS” in use case and click “Next”: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks-control-role.png)
   - Keep click “next” and at the final step input the role name: “eks-control-role” and click “Create Role”: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks-control-role-2.png)
7. Create IAM role “NodeInstanceRole”, which will be assumed by the EKS worker nodes:
   - Goto IAM in AWS console, select service: EC2 and click “Next”: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/nodeinstancerole.png)
   - Attach the following polices to the role:
     - AmazonEKSWorkerNodePolicy
     - AmazonEKS_CNI_Policy
     - AmazonEC2ContainerRegistryReadOnly
   - At the final step give the role a name “NodeInstanceRole” and create role: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/nodeinstancerole2.png)

8. Create EC2 key pair for the EKS worker node:
   - Goto EC2 in AWS console, click “key pair” at the left menu: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/ec2keypair.png)
   - Input the name and create key pair: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/ec2keypair2.png)


## Amazon EKS Cluster Setup
1. Goto “EKS” in AWS console, ensure selected region is Hong Kong: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks1.png)
2. Input the followings:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks2.png)
3. Select the pre-defined VPC and subnets for the EKS cluster:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks3.png)
4. Set the cluster security group, pick the default is fine:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks4.png)
5. Allow private access to API server endpoint only:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks5.png)
6. Set the below logging options and create the cluster:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks6.png)
7. Wait for successful creation of cluster, which means cluster status = “active”:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks7.png)
8. Scroll down to “Node Groups” and click “Add node group”:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks8.png)
9. Set node group configuration, ensure multiple subnets are selected:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks9.png)
10. Input SSH key pair for EKS worker nodes:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks10.png)
11. Set worker node compute configuration:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks11.png)
12.	Set scaling configuration:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks12.png)
13. Final step is review the group configuration and create, wait for the node group status become “active”:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks13.png)
14. Goto EC2 dashboard, there should be 3 EC2 instances running:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks14.png)
15. Now the EKS cluster setup is completed.

## Operation EC2 Instance Setup
An EC2 instance (aka. operation instance) will be setup at the public subnet of the EKS cluster's VPC, so that user from external networks can remote SSH to it and manage the EKS cluster using command line tool like eksctl, kubectl and helm.

1. Goto EC2 dashboard, click “launch instance”
2. Search “ami-03a9535798e343119”, this is a Linux AMI for a Bastion Host:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion2.png)
3. Select instance type , t3.micro should be fine:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion3.png)
4. Ensure a public subnet is selected:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion4.png)
5. Add 30GB storage:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion5.png)
6. Set security group, to allow SSH inbound traffic from internet:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion6.png)
7. Create a key-pair and launch the instance:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion7.png)
8. Check if the instance start successfully:![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/bastion8.png)
9. Click actions -> networking -> change security groups, add the instance to the EKS worker node security group as well.
10. Click the instance and click connect, follow the instruction there to SSH to the EC2 instance.
11. If fail to connect pls check the subnet’s route table if there is a route to internet gateway, also check if the EC2 has a public DNS name and IP.

## Setup AWS CLI, eksctl, kubectl and helm
1. The following steps are done in the operation EC2 instance.
2. Run the below command to install and upgrade AWS Command Line Tool (CLI)
`pip install awscli –upgrade –user`
3. Setup AWS account credential for CLI by “AWS Configure”:
`AWS Access Key ID [None]: <Your Access Key ID>`
`AWS Secret Access Key [None]: <Your Access Key Secret>`
`Default region name [None]: ap-east-1`
`Default output format [None]: json`
4. Download and extract the latest release of eksctl with the following command:
`curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp`
5. Move the extracted binary to /usr/local/bin:
`sudo mv /tmp/eksctl /usr/local/bin`
6. Test that your installation was successful with the following command:
`eksctl version`
7. Follow the below guide to setup kubectl, you should follow the guide for Linux and Kubernetes 1.14: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
8. To setup authentication between kubectl and the EKS cluster, use the AWS CLI update-kubeconfig command to create kubeconfig for your cluster, please replace “alfresco-prod” with your EKS cluster name:
`aws eks –region ap-east-1 update-kubeconfig –name alfresco-prod`
9. Test your configuration:
`kubectl get svc`
![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/kubectl9.png)
10.	Run the following command to install helm, a package manager on Kubernetes:
`curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > install-helm.sh`
11. And run:
`chmod u+x install-helm.sh`
`./install-helm.sh`
12. And finally run:
`helm init --service-account tiller`






