# Demonstrating ALB Ingress Controller

### Verify the Name of the Cluster with the CLI

```
cd ingress
aws eks list-clusters
```
### Create an IAM OIDC provider and associate it with your cluster

In 2014, AWS Identity and Access Management added support for federated identities using OpenID Connect (OIDC). This feature allows you to authenticate AWS API calls with supported identity providers and receive a valid OIDC JSON web token (JWT). You can pass this token to the AWS STS AssumeRoleWithWebIdentity API operation and receive IAM temporary role credentials. You can use these credentials to interact with any AWS service, like Amazon S3 and DynamoDB.

Kubernetes has long used service accounts as its own internal identity system. Pods can authenticate with the Kubernetes API server using an auto-mounted token (which was a non-OIDC JWT) that only the Kubernetes API server could validate. These legacy service account tokens do not expire, and rotating the signing key is a difficult process. In Kubernetes version 1.12, support was added for a new ProjectedServiceAccountToken feature, which is an OIDC JSON web token that also contains the service account identity, and supports a configurable audience.

With IAM roles for service accounts on Amazon EKS clusters, you can associate an IAM role with a Kubernetes service account. This service account can then provide AWS permissions to the containers in any pod that uses that service account. With this feature, you no longer need to provide extended permissions to the worker node IAM role so that pods on that node can call AWS APIs.

```
eksctl utils associate-iam-oidc-provider --cluster=rebrain-demo --approve
```

### Deploy RBAC Roles and RoleBindings needed by the AWS ALB Ingress controller
```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/rbac-role.yaml
kubectl apply -f rbac-role.yaml
```

### Download the AWS ALB Ingress controller YAML into a local file:
```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/alb-ingress-controller.yaml
```

### Edit the AWS ALB Ingress controller YAML to include the clusterName of the Kubernetes (or) Amazon EKS cluster.

Edit the –cluster-name flag to be the real name of our Kubernetes (or) Amazon EKS cluster in your alb-ingress-controller.yaml file. In this case, our cluster name was eksworkshop-eksctl as apparent from the output.

### Deploy the AWS ALB Ingress controller YAML:
```
kubectl apply -f alb-ingress-controller.yaml
```

### Verify that the deployment was successful and the controller started:
```
kubectl logs -n kube-system $(kubectl get po -n kube-system | grep alb-ingress | awk '{print $1}')
```

### Deploy 2048 game resources:
```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-namespace.yaml
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-deployment.yaml
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-service.yaml
kubectl apply -f 2048-namespace.yaml
kubectl apply -f 2048-deployment.yaml
kubectl apply -f 2048-service.yaml
```

### Deploy an Ingress resource for the 2048 game:
```
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-ingress.yaml
kubectl apply -f 2048-ingress.yaml
```

### Check
```
kubectl get ingress/2048-ingress -n 2048-game
```
### Check node port
```
kubectl describe service service-2048 --namespace=2048-game
```

### Cleanup
```
kubectl -n 2048-game delete pod,deployment,svc,ing --all
kubectl delete -f alb-ingress-controller.yaml
kubectl delete namespace 2048-game
```


