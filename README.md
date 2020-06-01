# Setup ssh keys and Github Configuration

```shell
vim ~/.gitconfig # Add your config
vim ~/.ssh/id_rsa # Add private key
chmod 600 ~/.ssh/id_rsa
vim ~/.ssh/id_rsa.pub # Add your public key
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys 
ssh-add
eval `ssh-agent`
 ssh-add
sudo yum install -y jq
```

# Extend FileSystem size

The default disk size for a cloud9 instance is 10gb but we need more than this for the tools needed to build ML projects.
So we need to increase the size of the EBS volume and grow the Linux filesytem. To do this use theaws 

```shell
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
```

# Create EKS Cluster

```shell
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export AWS_DEFAULT_REGION=$AWS_REGION
EKS_CLUSTER_NAME=mlops-c9

# Create KMS key for encrypted secrets support
key_arn=$(aws --region $AWS_DEFAULT_REGION kms create-key | jq -r '."KeyMetadata"["Arn"]')

mkdir -p ${EKS_CLUSTER_NAME}
pushd ${EKS_CLUSTER_NAME}
# Generate ssh key pair for access to EC2 nodes
ssh-keygen -b 4096 -f id_rsa -q -t rsa -N "" 2>/dev/null <<< y >/dev/null
curl  https://raw.githubusercontent.com/paulcarlton-ww/ml-workshop/master/resources/eks-template.yaml | \
    sed s/NAME/$EKS_CLUSTER_NAME/ | sed s/REGION/$AWS_DEFAULT_REGION/ | sed s#KEY#$key_arn# > eks.yaml

eksctl create cluster --config-file eks.yaml
popd
```
This will take about 20 minutes to complete, do 'kubectl get nodes' to verify it is ready

# Enable GitOps and deploy Application Development Profile
```shell
GITHUB_DIR=$PWD/src/github.com
GIT_EMAIL=$(git config -f ~/.gitconfig --get user.email)
GIT_ORG=$(git config -f ~/.gitconfig --get user.name)

EKSCTL_EXPERIMENTAL=true \
    eksctl enable repo \
        --git-url git@github.com:$GIT_ORG/${EKS_CLUSTER_NAME}-config \
        --git-email $GIT_EMAIL \
        --cluster $EKS_CLUSTER_NAME \
        --region "$AWS_DEFAULT_REGION"
```
Use flux ssh key to add deploy key to repo settings and give write access

Add [app-dev profile](https://eksctl.io/gitops-quickstart)

```shell
EKSCTL_EXPERIMENTAL=true eksctl enable profile app-dev \
        --git-url git@github.com:$GIT_ORG/${EKS_CLUSTER_NAME}-config \
        --git-email $GIT_EMAIL \
        --cluster $EKS_CLUSTER_NAME \
        --region "$AWS_DEFAULT_REGION"

cd $GITHUB_DIR/$GIT_ORG
git clone git@github.com:$GIT_ORG/${EKS_CLUSTER_NAME}-config.git
cd ${EKS_CLUSTER_NAME}-config
```
Wait a minute or two for flux to deploy applications, verify with 'kubectl get pods -A'

# Deploy Kubeflow

```shell
cd $GITHUB_DIR/$GIT_ORG
wget https://github.com/kubeflow/kfctl/releases/download/v1.0.2/kfctl_v1.0.2-0-ga476281_linux.tar.gz
tar -zxvf kfctl_v1.0.2-0-ga476281_linux.tar.gz 
mv kfctl ~/bin
rm kfctl_v1.0.2-0-ga476281_linux.tar.gz 

export AWS_CLUSTER_NAME=$EKS_CLUSTER_NAME
export KF_NAME=${AWS_CLUSTER_NAME}
export BASE_DIR=$GITHUB_DIR/$GIT_ORG/kubeflow-config
export KF_DIR=${BASE_DIR}/${KF_NAME}
mkdir -p ${KF_DIR}
cd ${KF_DIR}
export CONFIG_URI=https://raw.githubusercontent.com/kubeflow/manifests/v1.0-branch/kfdef/kfctl_aws.v1.0.2.yaml
wget -O kfctl_aws.yaml $CONFIG_URI
                                                                                                                                                                    
grep -v eksctl kfctl_aws.yaml > kfctl.yaml
sed -i s/kubeflow-aws/${AWS_CLUSTER_NAME}/ kfctl.yaml  
sed -i s/roles:/enablePodIamPolicy\:\ true/ kfctl.yaml  
sed -i s/us-west-2/${AWS_DEFAULT_REGION}/ kfctl.yaml  
sed -i s/roles:/enablePodIamPolicy\:\ true/ kfctl.yam

export CONFIG_FILE=${KF_DIR}/kfctl.yaml
kfctl apply -V -f ${CONFIG_FILE}
```

# Access the Kubeflow UI

Use terminal on your workstation (not cloud9 session)
```shell
export AWS_DEFAULT_REGION=eu-west-2 # the region from cloud9 environment
EKS_CLUSTER_NAME=mlops-c9 # name of your cluster
aws eks update-kubeconfig --name $EKS_CLUSTER_NAME
kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80 &
```

[Kubeflow UI](http://127.0.0.1:8080)
