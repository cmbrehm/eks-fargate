# EKS Demo

## Requirements
- EKS cluster created with public and private subnets
  - cluster configured for Fargate as per https://docs.aws.amazon.com/eks/latest/userguide/fargate-getting-started.html
- ECR instance created
- AWS Profile Configured
- bash shell/docker (can use Cloud9 IDE)

## Build and Push Docker

### Build Image
```
DEMO_ECR=$(aws ecr describe-repositories --profile AWSAdministratorAccess-123456789012 --query 'repositories[0].repositoryUri' --output text)
cd image
docker build -t $DEMO_ECR:demo .
```

### Test Locally
in the below command,
**8888** is the port that will be exposed on your machine.  **80** is the port exposed by the container

**You will need to know which
```
DEMO_CONTAINER=$(docker run -p 8888:80 -d $DEMO_ECR:demo)
curl http://localhost:8888/
docker stop $DEMO_CONTAINER
```

### Push To ECR
```
aws ecr get-login-password --region us-east-1 --profile AWSAdministratorAccess-123456789012 | docker login --username AWS --password-stdin $DEMO_ECR
docker push $DEMO_ECR:demo
```

### Important Outputs
- **Docker Image Name**: `$DEMO_ECR:demo`
- **Image Container Port**: `80`

## Deploy Pod To EKS
### Configure kubectl
you need to know cluster name use `aws eks list-clusters` to find it

```
EKS_CLUSTER=$(aws eks --region us-east-1 --profile AWSAdministratorAccess-332670553932 --query 'clusters[0]' --output=text list-clusters)
aws eks --region us-east-1 --profile AWSAdministratorAccess-332670553932 update-kubeconfig --name $EKS_CLUSTER
kubectl create namespace demo
```


### Create Fargate Profile for `demo` namespace
https://docs.aws.amazon.com/eks/latest/userguide/fargate-getting-started.html#fargate-gs-create-profile

Select only private subnets and use namespace `demo`


### Create Deployment File
this will config the standard template
```
cd k8s-deployment
sed 's@<IMAGE>@'$DEMO_ECR:demo'@' deployment.tmpl > deployment.yaml
kubectl apply -f deployment.yaml
```

### Validate
`kubectl get all -n demo` and verify pods are `Running`

#### Troubleshooting
https://aws.amazon.com/premiumsupport/knowledge-center/eks-pod-status-troubleshooting/
Delete and start again: `kubectl delete deployment demo-deployment -n demo`

## NodePort
```
cd nodeport
kubectl apply -f nodeport.yaml
kubectl get service/demo-nodeport-service -n demo
```

### Validate
```
kubectl get nodes -n demo -o wide

NAME                                   STATUS   ROLES    AGE   VERSION              INTERNAL-IP    EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
fargate-ip-10-100-44-1.ec2.internal    Ready    <none>   13m   v1.18.8-eks-7c9bda   10.100.44.1    <none>        Amazon Linux 2   4.14.203-156.332.amzn2.x86_64   containerd://1.3.2
fargate-ip-10-100-94-30.ec2.internal   Ready    <none>   13m   v1.18.8-eks-7c9bda   10.100.94.30   <none>        Amazon Linux 2   4.14.203-156.332.amzn2.x86_64   containerd://1.3.2
```
Get the INTERNAL-IP values from above you should be able to invoke `http://10.100.44.1:80/` AND `http://10.100.94.30:80/`.  This address will be on a private subnet so use Cloud9 IDE or a bastion EC2

## Load Balancer

For ALB on Fargate, you need to install the `alb-ingress-controller`

```
cd alb-controller
ALB_POLICY=$(aws iam create-policy --policy-name AWSLoadBalancerControllerPolicy2 --policy-document file://iam-policy.json --query 'Policy.Arn' --output text)
OIDC_PROVIDER=$(aws eks describe-cluster --name $EKS_CLUSTER --query "cluster.identity.oidc.issuer" --output text | sed -e "s/^https:\/\///")
sed 's/AWS_ACCOUNT_ID/'$AWS_ACCOUNT_ID'/' trust-policy.tmpl | sed 's@OIDC_PROVIDER@'$OIDC_PROVIDER'@' > trust.json
ALB_ROLE=$(aws iam create-role --role-name eksAlbRole --assume-role-policy-document file://trust.json --description "Role for EKS Load Balancer Controller" --query 'Role.Arn' --output text)
aws iam attach-role-policy --role-name eksAlbRole --policy-arn=$ALB_POLICY
kubectl create serviceaccount -n kube-system aws-load-balancer-controller
kubectl annotate serviceaccount -n kube-system aws-load-balancer-controller eks.amazonaws.com/role-arn=$ALB_ROLE
```

```
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
helm repo add eks https://aws.github.io/eks-charts
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller   --set clusterName=$EKS_CLUSTER   --set serviceAccount.create=false   --set serviceAccount.name=aws-load-balancer-controller -n kube-system
```



## Private Load Balancer

**NOTE:** private subnets need to be tagged per https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html#vpc-subnet-tagging

See this for additional annotations: https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer



```
cd lb-private
kubectl apply -f lbprivate.yaml
```

### Validate
```
kubectl describe service/private-lb -n demo


Name:                     private-lb
Namespace:                demo
Labels:                   app=demo
Annotations:              service.beta.kubernetes.io/aws-load-balancer-internal: true
Selector:                 app=demo
Type:                     LoadBalancer
IP:                       172.20.7.91
LoadBalancer Ingress:     internal-a289c041ac444432cb5aab278d4e3d33-1215845116.us-east-1.elb.amazonaws.com
Port:                     http  80/TCP
TargetPort:               80/TCP
NodePort:                 http  31131/TCP
Endpoints:                10.100.44.1:80,10.100.94.30:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  EnsuringLoadBalancer  77s   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   75s   service-controller  Ensured load balancer
```

- Look for `EnsuredLoadBalancer`.
- `curl http://internal-xxxxxx`

### Troubleshooting
https://aws.amazon.com/premiumsupport/knowledge-center/eks-load-balancers-troubleshooting/

helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=$EKS_CLUSTER \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.repository=918309763551.dkr.ecr.cn-north-1.amazonaws.com.cn/amazon/aws-load-balancer-controller \
  -n kube-system
