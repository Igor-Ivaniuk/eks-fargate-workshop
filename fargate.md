# Demonstrating AWS Fargate for EKS

### Create a Fargate profile
```
eksctl create fargateprofile --cluster rebrain-demo --name 2048-game --namespace 2048-game
```

### Create a Fargate profile
```
eksctl create fargateprofile --cluster rebrain-demo --name 2048-game --namespace 2048-game
eksctl get fargateprofile --cluster rebrain-demo -o yaml
```

### Deploy 2048 game resources:
```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-namespace.yaml
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-deployment.yaml
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.5/docs/examples/2048/2048-service.yaml
kubectl apply -f 2048-namespace.yaml
kubectl apply -f 2048-deployment.yaml
kubectl apply -f 2048-service.yaml
```

### Add resources requests to deployment
```
    resources:
      requests:
        memory: "64Mi"
        cpu: "4"
      limits:
        memory: "128Mi"
        cpu: "4"
```

### Create an IAM Policy for ALB Ingress
```
wget https://eksworkshop.com/beginner/180_fargate/fargate.files/alb-ingress-iam-policy.json
aws iam create-policy --policy-name ALBIngressControllerIAMPolicy --policy-document file://alb-ingress-iam-policy.json
```

### Creating Service Account
```
export FARGATE_POLICY_ARN=$(aws iam list-policies --query 'Policies[?PolicyName==`ALBIngressControllerIAMPolicy`].Arn' --output text)
eksctl create iamserviceaccount --name alb-ingress-controller --namespace 2048-game --cluster rebrain-demo --attach-policy-arn ${FARGATE_POLICY_ARN} --approve --override-existing-serviceaccounts
kubectl get sa alb-ingress-controller -n 2048-game -o yaml
```

### Create RBAC Role
```
wget https://eksworkshop.com/beginner/180_fargate/fargate.files/rbac-role.yaml
kubectl apply -f rbac-role.yaml
```

### Deploy ALB Ingress Controller
```
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
export VPC_ID=$(aws eks describe-cluster --name rebrain-demo --query "cluster.resourcesVpcConfig.vpcId" --output text)
helm --namespace 2048-game install 2048-game \
  incubator/aws-alb-ingress-controller \
  --set image.tag=v1.1.4 \
  --set awsRegion=${AWS_REGION} \
  --set awsVpcID=${VPC_ID} \
  --set rbac.create=false \
  --set rbac.serviceAccount.name=alb-ingress-controller \
  --set clusterName=rebrain-demo
kubectl -n 2048-game rollout status deployment 2048-game-aws-alb-ingress-controller
```

### Deploying ingress
```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.5/docs/examples/2048/2048-ingress.yaml

add annotation  alb.ingress.kubernetes.io/target-type: ip
(or use command below)
yq w -i 2048-ingress.yaml 'metadata.annotations."alb.ingress.kubernetes.io/target-type"' ip

kubectl apply -f 2048-ingress.yaml
```

### Get the HTTP domain
```
export ALB_ADDRESS=$(kubectl get ingress -n 2048-game -o json | jq -r '.items[].status.loadBalancer.ingress[].hostname')
echo "http://${ALB_ADDRESS}"
```

### Cleanup
```
kubectl delete -f 2048-ingress.yaml
helm -n 2048-game delete 2048-game
kubectl -n 2048-game delete pod,deployment,svc,ing --all
kubectl delete namespace 2048-game
eksctl delete fargateprofile --cluster rebrain-demo --name 2048-game
```