Setup

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

export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export AWS_DEFAULT_REGION=$AWS_REGION
EKS_CLUSTER_NAME=mlops-c9
GITHUB_DIR=$PWD/src/github.com
GIT_EMAIL=$(git config -f ~/.gitconfig --get user.email)
GIT_ORG=$(git config -f ~/.gitconfig --get user.name)

key_arn=$(aws --region $AWS_DEFAULT_REGION kms create-key | jq -r '."KeyMetadata"["Arn"]')
mkdir -p ${EKS_CLUSTER_NAME}
pushd ${EKS_CLUSTER_NAME}
ssh-keygen -b 4096 -f id_rsa -q -t rsa -N "" 2>/dev/null <<< y >/dev/null
curl  https://raw.githubusercontent.com/paulcarlton-ww/ml-workshop/master/resources/eks-template.yaml | \
    sed s/NAME/$EKS_CLUSTER_NAME/ | sed s/REGION/$AWS_DEFAULT_REGION/ | sed s#KEY#$key_arn# > eks.yaml

eksctl create cluster --config-file eks.yaml
popd

EKSCTL_EXPERIMENTAL=true \
    eksctl enable repo \
        --git-url git@github.com:$GIT_ORG/${EKS_CLUSTER_NAME}-config \
        --git-email $GIT_EMAIL \
        --cluster $EKS_CLUSTER_NAME \
        --region "$AWS_DEFAULT_REGION"

# Use flux ssh key to add deploy key to repo settings and give write access

# Add app-dev profile see https://eksctl.io/gitops-quickstart/

EKSCTL_EXPERIMENTAL=true eksctl enable profile app-dev \
        --git-url git@github.com:$GIT_ORG/${EKS_CLUSTER_NAME}-config \
        --git-email $GIT_EMAIL \
        --cluster $EKS_CLUSTER_NAME \
        --region "$AWS_DEFAULT_REGION"

cd $GITHUB_DIR/$GIT_ORG
git clone git@github.com:$GIT_ORG/${EKS_CLUSTER_NAME}-config.git
cd ${EKS_CLUSTER_NAME}-config

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

kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80 &

aws iam create-user --user-name mlops-user
aws iam create-access-key --user-name mlops-user > mlops-user.json
export THE_ACCESS_KEY_ID=$(jq '."AccessKey"["AccessKeyId"]' mlops-user.json)
echo $THE_ACCESS_KEY_ID
export THE_SECRET_ACCESS_KEY=$(jq '."AccessKey"["SecretAccessKey"]' mlops-user.json)

export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)

aws iam create-policy --policy-name mlops-s3-access \
    --policy-document https://raw.githubusercontent.com/paulcarlton-ww/ml-workshop/master/resources/s3-policy.json  > s3-policy.json

aws iam create-policy --policy-name mlops-emr-access \
    --policy-document https://raw.githubusercontent.com/paulcarlton-ww/ml-workshop/master/resources/emr-policy.json > emr-policy.json

aws iam create-policy --policy-name mlops-iam-access \
    --policy-document https://raw.githubusercontent.com/paulcarlton-ww/ml-workshop/master/resources/iam-policy.json > iam-policy.json

aws iam attach-user-policy --user-name mlops-user  --policy-arn $(jq '."Policy"["Arn"]' s3-policy.json)
aws iam attach-user-policy --user-name mlops-user  --policy-arn $(jq '."Policy"["Arn"]' emr-policy.json)

curl  https://raw.githubusercontent.com/paulcarlton-ww/ml-workshop/master/resources/kubeflow-aws-secret.yaml | \
    sed s/YOUR_BASE64_SECRET_ACCESS/$(echo -n "$THE_SECRET_ACCESS_KEY" | base64)/ | \
    sed s/YOUR_BASE64_ACCESS_KEY/$(echo -n "$THE_ACCESS_KEY_ID" | base64)/ | kubectl apply -f -;echo

aws s3api create-bucket --bucket mlops-kubeflow-pipeline-data --region eu-west-2 --create-bucket-configuration LocationConstraint=eu-west-2


