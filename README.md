# Karpenter on Existing EKS Cluster
 
## Karpenter Installation Script (For Harness Pipeline)
```
#!/bin/bash

# Update packages
apt update -y

# Install AWS CLI if not already installed
if ! command -v aws &> /dev/null; then
    echo "Installing AWS CLI..."
    apt install curl -y && apt install unzip
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    ./aws/install
else
    echo "AWS CLI is already installed."
fi

# Install Kubectl if not already installed
if ! command -v kubectl &> /dev/null; then
    echo "Installing Kubectl..."
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
else
    echo "Kubectl is already installed."
fi

# Install EKSCTL if not already installed
if ! command -v eksctl &> /dev/null; then
    echo "Installing EKSCTL..."
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    mv /tmp/eksctl /usr/local/bin
else
    echo "EKSCTL is already installed."
fi

# Install Helm if not already installed
if ! command -v helm &> /dev/null; then
    echo "Installing Helm..."
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
else
    echo "Helm is already installed."
fi

# Configure AWS CLI credentials
aws configure set aws_access_key_id "<+pipeline.variables.secret_key>"
aws configure set aws_secret_access_key "<+pipeline.variables.secret_access_key>"
aws configure set region "us-west-2"
aws configure set output "json"

export AWS_PAGER=""

# Check if EKS cluster exists before listing
if aws eks describe-cluster --name terraform-eks-cluster-poc > /dev/null 2>&1; then
    aws eks update-kubeconfig --name terraform-eks-cluster-poc
    kubectl get no
else
    echo "EKS cluster not found."
fi

CLUSTER_NAME=terraform-eks-cluster-poc
KARPENTER_VERSION="1.0.6"
TEMPOUT="$(mktemp)"
KARPENTER_NAMESPACE="kube-system"
KARPENTER_SERVICE_ACCOUNT="karpenter"

# Attach Tags to Security Group and Subnets for service discovery
SEC_GROUP_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --query cluster.resourcesVpcConfig.clusterSecurityGroupId --output text)
if [[ -n "$SEC_GROUP_ID" ]]; then
    echo "Attaching tags to security group and subnets..."
    aws ec2 create-tags --resources $SEC_GROUP_ID --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME

    SUBNET_IDS=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.resourcesVpcConfig.subnetIds" --output text | tr '\t' ' ')
    IFS=' ' read -r -a SUBNET_IDS_ARRAY <<< "$SUBNET_IDS"
    for subnet in "${SUBNET_IDS_ARRAY[@]}"; do
        aws ec2 create-tags --resources "$subnet" --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME
    done
else
    echo "Cluster security group not found."
fi

# Deploy CloudFormation stack if not already deployed
if ! aws cloudformation describe-stacks --stack-name "Karpenter-${CLUSTER_NAME}" > /dev/null 2>&1; then
    echo "Deploying CloudFormation stack..."
    curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml  > "${TEMPOUT}" \
    && aws cloudformation deploy --stack-name "Karpenter-${CLUSTER_NAME}" --template-file "${TEMPOUT}" --capabilities CAPABILITY_NAMED_IAM --parameter-overrides "ClusterName=${CLUSTER_NAME}"
else
    echo "CloudFormation stack already deployed."
fi

sleep 30s

# Set IAM role and other configurations
KARPENTER_NODE_ROLE=KarpenterNodeRole-${CLUSTER_NAME}
AWS_PARTITION="aws"
AWS_REGION="us-west-2"
OIDC_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text --region ${AWS_REGION})"
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text --region ${AWS_REGION})

# Create IAM role for Karpenter if it does not exist
if ! aws iam get-role --role-name KarpenterControllerRole-${CLUSTER_NAME} > /dev/null 2>&1; then
    echo "Creating IAM role for Karpenter..."
    cat <<EOF > controller-trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ENDPOINT#*//}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${OIDC_ENDPOINT#*//}:aud": "sts.amazonaws.com",
                    "${OIDC_ENDPOINT#*//}:sub": "system:serviceaccount:${KARPENTER_NAMESPACE}:${KARPENTER_SERVICE_ACCOUNT}"
                }
            }
        }
    ]
}
EOF

    aws iam create-role --role-name KarpenterControllerRole-${CLUSTER_NAME} --assume-role-policy-document file://controller-trust-policy.json --region ${AWS_REGION}
else
    echo "IAM role for Karpenter already exists."
fi


# Attach policy to IAM role
POLICY_NAME=KarpenterControllerPolicy-${CLUSTER_NAME}
POLICY_ARN=$(aws iam list-policies --query "Policies[?PolicyName=='${POLICY_NAME}'].{ARN:Arn}" --output text)
aws iam attach-role-policy --role-name KarpenterControllerRole-${CLUSTER_NAME} --policy-arn "$POLICY_ARN"

# Karpenter installation
if ! kubectl get sa -n ${KARPENTER_NAMESPACE} ${KARPENTER_SERVICE_ACCOUNT} > /dev/null 2>&1; then
    echo "Installing Karpenter..."
    kubectl create sa -n ${KARPENTER_NAMESPACE} ${KARPENTER_SERVICE_ACCOUNT}
    kubectl annotate sa -n ${KARPENTER_NAMESPACE} ${KARPENTER_SERVICE_ACCOUNT} eks.amazonaws.com/role-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole-${CLUSTER_NAME}
    
    eksctl create iamidentitymapping --username system:node:{{EC2PrivateDNSName}} --cluster ${CLUSTER_NAME} --arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}" --group system:bootstrappers --group system:nodes

    helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
      --set "serviceAccount.create=false" \
      --set "serviceAccount.name=${KARPENTER_SERVICE_ACCOUNT}" \
      --set "settings.clusterName=${CLUSTER_NAME}" \
      --set "settings.interruptionQueue=${CLUSTER_NAME}" \
      --set controller.resources.requests.cpu=1 \
      --set controller.resources.requests.memory=1Gi \
      --set controller.resources.limits.cpu=1 \
      --set controller.resources.limits.memory=1Gi \
      --wait
else
    echo "Karpenter is already installed."
fi
```

## NodePool and EC2NodeClass Creation

```
CLUSTER_NAME=terraform-eks-cluster-poc
KARPENTER_NODE_ROLE=KarpenterNodeRole-${CLUSTER_NAME}

## AWS CLI
apt update -y
apt install curl -y && apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
aws --version

## Install Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

aws configure set aws_access_key_id "<+pipeline.variables.secret_key>" && \
aws configure set aws_secret_access_key "<+pipeline.variables.secret_access_key>" && \
aws configure set region "us-west-2" && \
aws configure set output "json"

export AWS_PAGER=""
aws eks list-clusters
aws eks update-kubeconfig --name ${CLUSTER_NAME}
kubectl get no

apt-get install gettext -y

cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - t3.micro
            - t3.small
            - t3.medium
            - t3.large
            - t3.xlarge
            - t3.2xlarge
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h # 30 * 24h = 720h
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # Required, resolves a default ami and userdata
  amiFamily: AL2 # Amazon Linux 2
  role: "${KARPENTER_NODE_ROLE}" # replace with your cluster name
  # Required, discovers subnets to attach to instances
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  # Required, discovers security groups to attach to instances
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
EOF
```