# AWS-Immersion-Day for CNF Partner/Customer in Telco

## 1. Please download CFN template for this session
* CFN repo: https://github.com/crosscom/AWS-Immersion-Day/blob/main/template/aws-immersion-infra.yaml
* This creates VPC, public/private subnets, subnet route tables, IGW, NAT-GW, Security Groups, EKS Cluster and Bastion Instance

## 2. Log in to AWS Event Engine 

* https://dashboard.eventengine.run/dashboard.
* Please put Event Hash (will be given at the session from your instructor). 
* Please use OTP authentication with your company email.
* Click "Set Team Name" (Team is equivalent to the account)
    * Please put TEAM name to be your name.  
* Click "SSH key" 
    * Download **ee-default-keypair** to your PC (just in case, copy key material to notepad as well).
* Click "AWS Console"
    * Copy credentials (export AWS_DEFAULT_REGION=..) 
    * Click "Open AWS Console".
![Dashboard](image/dashboard.png)

## 3. Create Environment with CloudFormation
* Type "CloudFormation" at search service section and go to CloudFormation.
* Create Stack -> upload a template file -> Choose file (select downloaded "nokia-immersion-infra.yaml").


![Landing Zone configuratoin](image/immersion-day1.png)


## 4. Login to Bastion Host 
* Usually in eksworkshop or normal immersionday offered by AWS, we guide customer to experience Cloud9 (AWS IDE environment). But in this workshop, plan is to provide a general environment with your own Bastion Host EC2, where you have to install kubectl tools and other tools as needed.
* (General)
    * We can use EC2 Instance Connect to login to EC2 instance.
    * EC2->Instances->"connect" (right top corner of screen). 
    * click "connect"

* (MAC user) Log in from your laptop
    * Let's use key pair we downloaded to access to the instance.

  ````
  chmod 600 ee-default-keypair.pem
  ssh-add ee-default-keypair.pem
  ssh -A ec2-user@54.208.182.244
  ````

    * Copy AWS credentials (AWS_DEFAULT_REGION, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)

  ````
  export AWS_DEFAULT_REGION=us-east-1
  export AWS_ACCESS_KEY_ID=ASIA..
  export AWS_SECRET_ACCESS_KEY=4wyDA..
  export AWS_SESSION_TOKEN=IQo...
  ````

    * Try whether AWS confidential is already configured well

    ````
    aws sts get-caller-identity
    {
      "Account": "400166681888", 
      "UserId": "AROAV2K6K7UQPEU2EMBY3:MasterKey", 
      "Arn": "arn:aws:sts::400166681888:assumed-role/TeamRole/MasterKey"
    }
    ````

* (Window user) Log in from your laptop
    * https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html

## 5. Make a Bastion Host to be a kubectl client

* Download kubectl. 

  ````
  curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
  curl -o kubectl.sha256 https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/``amd64``/kubectl.sha256
  openssl sha1 -sha256 kubectl
  chmod +x ./kubectl
  mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
  echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
  kubectl version —short —client
  ````

* Check your name of EKS cluster (from CloudFormation output or EKS console (service search -> EKS))

* Config kubeconfig with EKS CLI
  ````
  aws eks update-kubeconfig --name=eks-my-first-stack
  ````

* Verify kubectl command
  ````
  kubectl get svc
  NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
  kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   31m
  ````

* Verify it from AWS CLI
  ````
  aws eks describe-cluster --name eks-my-first-stack
  ````

## 6. Self-managed Node Group creation (with Multus CNI Plugin)
* Go to S3 and create bucket (folder/directory) with *Create bucket*.
* Bucket name to be unique like young-jung-immersion, and then *Create bucket*.
* Click the bucket you just created and drag & drop lambda_function.zip file (which you can find from /template directory of this GitHub). Then, click *Upload*.
* Please memorize bucket name you create (this is required in CloudFormation)
* Go to CloudFormation console by selecting CloudFormation from Services drop down or by search menu. 
    * Select *Create stack*, *with new resources(standard)*.
    * Click *Template is ready" (default), "Upload a template file", "Choose file". Select "amazon-eks-nodegroup-multus.yaml" file that you have downloaded from this GitHub. 
    * Stack name -> ng1
    * ClusterName -> eks-immersion (your own name)
    * ClusterControlPlaneSecurityGroup -> "immersion-EksControlSecurityGroup-xxxx"
    * NodeGroupName -> ng1
    * Min/Desired/MaxSize -> 1/1/1
    * KeyName -> ee-default-keypair
    * VpcId -> vpc-immersion (that you created)
    * Subnets -> privateAz1-immersion (this is for main primary K8s networking network)
    * MultusSubnets -> multus1Az1 and Multus2Az1
    * MultusSecurityGroups -> immersion-MultusSecurityGroup
    * LambdaS3Bucket -> the one you created (young-jung-immersion)
    * LambdaS3Key -> lambda_function.zip
    * *Next*, check "I acknowledge...", and then *Next*.

* Once CloudFormation stack creation is completed, check *Output* part in the menu and copy the value of NodeInstanceRole (e.g. arn:aws:iam::153318889914:role/ng1-NodeInstanceRole-1C77OUUUP6686)
* Go to the Bastion Host where we can run kubectl command. 
* Download aws-auth-cm file at Bastion Host.
  ````
  curl -o aws-auth-cm.yaml https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/aws-auth-cm.yaml
  ````

* Open aws-auth-cm.yaml file downloaded using vi or any text editor. And place above copied NodeInstanceRole value to the place of "*<ARN of instance role (not instance profile)>*", and then apply this through kubectl.
  ````
  kind: ConfigMap
  metadata:
    name: aws-auth
    namespace: kube-system
  data:
    mapRoles: |
      - rolearn: arn:aws:iam::153318889914:role/ng1-NodeInstanceRole-1C77OUUUP6686
        username: system:node:{{EC2PrivateDNSName}}
        groups:
          - system:bootstrappers
          - system:nodes
  ````
  ````
  kubectl -f apply aws-auth-cm.yaml
  ````

## 7. EKS-managed Node Group 
* Let's try to create EKS-managed node group now.
* Go to EKS console (service search -> type EKS -> select *Elastic Kubernetes Service*)
* Select *Clusters* under Amazon EKS from left side of control pane. 
* Click your EKS cluster in the list, and select *Configuration*.
* Select *Compute*, and then *Add Node Group*. 
* Fill *Name*->ng2
* Select *Node IAM Role* -> the one already created in Step 7 by CFN. 
* *Min/Desired/MaxSize* -> 1/1/1
* *Subnets* -> privateAz1
* *SSH Key pair* -> ee-default-keypair
* Click *Create* at Review and create page. 
* After the creation of EKS managed node group (ng2), it can be verified in kubectl as 2 worker nodes in the cluster (each from ng1 (self-managed) and ng2 (EKS managed).
  ````
  kubectl get node
  NAME                        STATUS   ROLES    AGE     VERSION
  ip-10-0-2-33.ec2.internal   Ready    <none>   8m22s   v1.19.6-eks-49a6c0
  ip-10-0-2-47.ec2.internal   Ready    <none>   18m     v1.19.6-eks-49a6c0
  ````

## 8. Clean up environment
* Delete Node Group in EKS menu. 
* Go to CloudFormation and Delete ng1 stack. 
* After completion of above, delete the first infra stack. 



