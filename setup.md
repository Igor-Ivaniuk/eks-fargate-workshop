# Installing tools

## Kubectl

```
sudo curl --location -o /usr/local/bin/kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.16.8/2020-04-16/bin/linux/amd64/kubectl
sudo chmod +x /usr/local/bin/kubectl
```

## awscli

```
sudo pip install --upgrade awscli && hash -r
```

## Bash tools
```
sudo yum -y install jq gettext bash-completion moreutils
echo 'yq() {
  docker run --rm -i -v "${PWD}":/workdir mikefarah/yq yq "$@"
}' | tee -a ~/.bashrc && source ~/.bashrc
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
```

## kubectl bash completion
```
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

## IAM Role
WorkenvAdminRole

arn:aws:iam::637530424188:role/WorkenvAdminRole

## SSH key
```
ssh-keygen
aws ec2 import-key-pair --key-name "workenv_key" --public-key-material file://~/.ssh/id_rsa.pub
```

## KMS key
```
aws kms create-alias --alias-name alias/workenv --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
export MASTER_ARN=$(aws kms describe-key --key-id alias/workenv --query KeyMetadata.Arn --output text)
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
```

## eksctl
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
eksctl version
```

## Create cluster

```
cat << EOF > cluster.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: rebrain-demo
  region: ${AWS_REGION}

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 2
  instanceType: t3.medium
  iam:
    withAddonPolicies:
      albIngress: true

# fargateProfiles:
#   - name: fp-default
#     selectors:
#       # All workloads in the "default" Kubernetes namespace will be
#       # scheduled onto Fargate:
#       - namespace: default
#       # All workloads in the "kube-system" Kubernetes namespace will be
#       # scheduled onto Fargate:
#       - namespace: kube-system
#   - name: fp-dev
#     selectors:
#       # All workloads in the "dev" Kubernetes namespace matching the following
#       # label selectors will be scheduled onto Fargate:
#       - namespace: dev
#         labels:
#           env: dev

secretsEncryption:
  keyARN: ${MASTER_ARN}
EOF
```

```
eksctl create cluster -f cluster.yaml
```

## Install the Helm CLI
```
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)
```