1.) Download AWS CLI and kubectl
2.) Create AWS KEY
3.) AWS Configure
4.) Region ex: us-east-2
5.) Create IAM role

 cat >eks-cluster-role-trust-policy.json <<EOF 
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

# CREATE ROLE BASE ON ABOVE FILE
 aws iam create-role --role-name myAmazonEKSClusterRole --assume    -role-policy-document file://"eks-cluster-role-trust-policy.json"

# GET INFORMATION ABOUT CREATED ROLE
aws iam get-role --role-name myAmazonEKSClusterRole

# CHECK POLICIES > SHOULD BE 0
aws iam list-attached-role-policies --role-name myAmazonEKSClusterRole

# ADD NEW POLICIES
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy --role-name myAmazonEKSClusterRole

6) CREATE FREE VPC FROM AWS SITE
curl https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-sample.yaml
# PUSH TEMPLATE BY THE USE OF CLOUDFORMATION
aws cloudformation deploy --template-file amazon-eks-vpc-sample.yaml --stack-name myAmazonEKSClusterRole --region us-east-2
# LIST ALL RESOURCES UNDER CREATED STACK
aws cloudformation list-stack-resources --stack-name myAmazonEKSClusterRole --region us-east-2 
# BELOW INFORMATIONS WILL BE NEEDED FOR NEXT STEP,CLUSTER WILL BE PLACED INTO THIS VPC
PhysicalResourceID:subnet-08860147XXXXXXXXX
PhysicalResourceID:subnet-0f64e73dXXXXXXXXX
PhysicalResourceID:subnet-0365f41aXXXXXXXXX
PhysicalResourceId:sg-0a76349f98XXXXXXXX,


7) CREATE CLUSTER
aws eks create-cluster --name myAmazonEKSCluster --role-arn arn:aws:iam::4533XXXXXXX:role/myAmazonEKSClusterRole --resources-vpc-config subnetIds=subnet-08860147dXXXXXX,subnet-0f64e73XXXXXX,subnet-0365f41XXXXXXXX,securityGroupIds=sg-0a76XXXXXXXXXX,endpointPublicAccess=true,endpointPrivateAccess=false --region us-east-2
# LIST CLUSTERS
 aws eks list-clusters --region us-east-2


8) POINT TO AWS CLI YOUR CLUSTER,BELOW CONFIG WILL DOWNLOAD CONFIG AND SAVE IT LOCALLY UNDER ~/.kube/config
aws eks update-kubeconfig --name myAmazonEKSClusterRole --region us-east-2
cat ~/.kube/config


INFO > BETWEEN CONFIGURATIONS YOU CAN SWITCH BY THE USE OF
kubectl config use-context


9) kubectl get nodes >>> SHOULD SHOW NOTHING, ONLY CONTROL PLANE CREATED FOR NOW

10)NODE GROUP CREATION

#ROLE CREATION

  cat >assume-node-policy.json <<EOF 
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
       }
    ]
  }

# CREATE ROLE FOR WORKER NODES BASED ON ABOVE FILE >  myAmazonEKSClusterRole-NODES
role_arn=$(aws iam create-role --role-name myAmazonEKSClusterRole-NODES --assume-role-policy-document file://"assume-node-policy.json")

# GET INFORMATION ABOUT CREATED ROLE
aws iam get-role --role-name myAmazonEKSClusterRole-NODE

# ADD 3 POLICIES
aws iam attach-role-policy --role-name myAmazonEKSClusterRole-NODES --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name myAmazonEKSClusterRole-NODES --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name myAmazonEKSClusterRole-NODES --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
# CHECK POLICIES
aws iam list-attached-role-policies --role-name  myAmazonEKSClusterRole-NODES


# INFORMATION ABOUT CREATED CLUSTER
clustername= myAmazonEKSCluster
nodegroupName=test
node-role=$role_arn
subnets=subnet-088601XXXXXXXXX


# NODE GROUP CREATION
aws eks create-nodegroup \
--cluster-name myAmazonEKSCluster \
--nodegroup-name test \
--node-role arn:aws:iam::453309330431:role/myAmazonEKSClusterRole-NODES \
--subnets subnet-08860147dXXXXXX \
--disk-size 200 \
--scaling-config minSize=1,maxSize=2,desiredSize=1 \
--instance-type t2.small --region us-east-2

# GO TO AWS  AWS>EKS>Cluster > You see NODEGROUPS CREATED

11) kubectl get nodes > SHOULD SHOW ALL CREATED NODES

12) DEPLOYMENT
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml

kubectl get svc
GO TO LOADBALANCER URL >> a0dbecf7693634206a29643XXXXXX-703XXXXX.us-east-2.elb.amazonaws.com


13) CLEANUP

# CLUSTER INFOS
clustername= myAmazonEKSCluster
nodegroup= test

# DELETE NODEGROUP
aws eks delete-nodegroup --cluster-name myAmazonEKSCluster --nodegroup-name test --region us-east-2
# DELETE CLUSTER
aws eks delete-cluster --name  myAmazonEKSClusterRole --region us-east-2

# REMOVE CLUSTER POLICY AND ROLE
aws iam detach-role-policy --role-name myAmazonEKSClusterRole --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
aws iam delete-role --role-name myAmazonEKSClusterRole

# REMOVE NODE POLICIES AND ROLE
aws iam detach-role-policy --role-name myAmazonEKSClusterRole-NODES --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam detach-role-policy --role-name myAmazonEKSClusterRole-NODES --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy 
aws iam detach-role-policy --role-name myAmazonEKSClusterRole-NODES --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly



#EKSCTL
#EKSCTL WILL USE CLOUDFORMATION
#DEFAULT VPC WILL BE USED IF NOT DEPICTED
eksctl create cluster --name myAmazonEKSCluster \
--region us-east-2 \
--version 1.16 \
--managed \
--node-type t2.small \
--nodes 1 \
--node-volume-size 200


#CLEANUP EKSCTP
aws cloudformation delete-stack --stack-name myAmazonEKSCluster
