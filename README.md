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
	a. Goto IAM in AWS console, select service: EKS and click “EKS” in use case and click “Next”: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks-control-role.png)
	b. Keep click “next” and at the final step input the role name: “eks-control-role” and click “Create Role”: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/eks-control-role-2.png)
7. Create IAM role “NodeInstanceRole”, which will be assumed by the EKS worker nodes:
  a. Goto IAM in AWS console, select service: EC2 and click “Next”: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/nodeinstancerole.png)
  b. Attach the following polices to the role:
   i. AmazonEKSWorkerNodePolicy
   ii. AmazonEKS_CNI_Policy
   iii. AmazonEC2ContainerRegistryReadOnly
  c. At the final step give the role a name “NodeInstanceRole” and create role: ![](https://raw.githubusercontent.com/peterone928/alfresco-eks/master/images/nodeinstancerole2.png)



