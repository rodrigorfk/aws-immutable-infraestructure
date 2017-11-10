# Requisitos
Na lista abaixo é possivel encontrar os requisitos necessrios para rodar os exemplos aqui contidos:
* [`python pip`](https://pip.pypa.io/en/stable/installing/)
* [`AWS CLI`](http://docs.aws.amazon.com/pt_br/cli/latest/userguide/installing.html)
* [`packer`](https://www.packer.io/intro/getting-started/install.html)
* [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [`kops`](https://github.com/kubernetes/kops/blob/master/docs/install.md)
* [`Ruby`](https://www.ruby-lang.org/pt/documentation/installation/)
* [`jq`](https://github.com/stedolan/jq/wiki/Installation)

# Exemplo completo
O exemplo completo da solução aqui demostrada está no arquivo [guideline.sh](guideline.sh)

# Exemplo detalhado
Segue abaixo a descrição detalhada do exemplo:
## Configuração das credenciais básicas da sua conta AWS
```shell
aws configure --profile my-aws-admin-user
export AWS_PROFILE=my-aws-admin-user
export AWS_DEFAULT_REGION=us-west-2
```
## Criação de uma stack no cloudformation para obter credenciais IAM para rodar o Packer e o KOPS
```shell
aws cloudformation create-stack --stack-name immutable-iam-permissions --template-body file://cfn-templates/iam-permisions.yaml --capabilities CAPABILITY_NAMED_IAM
aws cloudformation wait stack-create-complete --stack-name immutable-iam-permissions
aws iam create-access-key --user-name immutable-infraestructure-user > permissions.json
export AWS_ACCESS_KEY=$(cat permissions.json | jq .AccessKey.AccessKeyId --raw-output)
export AWS_SECRET_KEY=$(cat permissions.json | jq .AccessKey.SecretAccessKey --raw-output)
export AWS_ACCESS_KEY_ID=$(cat permissions.json | jq .AccessKey.AccessKeyId --raw-output)
export AWS_SECRET_ACCESS_KEY=$(cat permissions.json | jq .AccessKey.SecretAccessKey --raw-output)
```
## Faz o build da primeira AMI usando o Packer
```shell
cd packer/centos-base
packer validate packer-template.json
packer build packer-template.json 
```

## Faz o build da segunda AMI usando o Packer com base na AMI gerada anteriormente
```shell
export CENTOS_BASE_AMI=$(aws ec2 describe-images --owners self --filters Name=tag-key,Values=Name Name=tag-value,Values="CentOS Linux 7 AMI" --query 'Images[0].ImageId' --output text)
cd ../docker-base
packer validate -var aws_base_ami=$CENTOS_BASE_AMI packer-template.json
packer build -var aws_base_ami=$CENTOS_BASE_AMI packer-template.json
```
## Faz o build da terceira AMI
```shell
cd ../k8s-centos-base/
packer validate -var aws_base_ami=$CENTOS_BASE_AMI packer-template.json
packer build -var aws_base_ami=$CENTOS_BASE_AMI packer-template.json
export K8S_BASE_AMI=$(aws ec2 describe-images --owners self --filters Name=tag-key,Values=type Name=tag-value,Values=kubernetes --query 'Images[0].ImageId' --output text)
```
### Cria o Cluster do Kubernetes usando o Kops
```shell
cd ../..
aws s3api create-bucket --bucket my-kops-tdc2017-bucket --region $AWS_DEFAULT_REGION --create-bucket-configuration LocationConstraint=$AWS_DEFAULT_REGION
aws route53 list-hosted-zones

export KOPS_STATE_STORE=s3://my-kops-tdc2017-bucket
export VPC_ID=$(aws ec2 describe-vpcs --query 'Vpcs[0].VpcId' --output text)
export SUBNET_ID=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID Name=availabilityZone,Values=${AWS_DEFAULT_REGION}a --query 'Subnets[0].SubnetId' --output text)
export DNS_ZONE_PRIVATE_ID=Z29FJE9DGAE7L0
export NAME=k8s.tdc.kunit.com.br
ssh-keygen -t rsa -f ssh-key.pem

kops create cluster \
    --node-count 2 \
    --master-count 1 \
    --zones ${AWS_DEFAULT_REGION}a \
    --master-zones ${AWS_DEFAULT_REGION}a \
    --dns-zone=${DNS_ZONE_PRIVATE_ID} \
    --node-volume-size 30 \
    --master-volume-size 15 \
    --image ${K8S_BASE_AMI} \
    --dns public \
    --authorization RBAC \
    --cloud aws \
    --node-size t2.medium \
    --master-size t2.small \
    --topology public \
    --networking weave \
    --vpc=${VPC_ID} \
    --ssh-public-key ./ssh-key.pem.pub \
    ${NAME}

kops get cluster
kops get ig --name=$NAME

kops get clusters $NAME -o yaml > state.yaml
ruby -ryaml -rjson -e 'puts YAML.load($stdin.read).to_json' < state.yaml | jq --arg SUBNET_ID $SUBNET_ID '.spec.kubernetesVersion |= "1.7.0" | .spec.subnets[0].id |= $SUBNET_ID | del(.spec.subnets[0].cidr)  | del(.metadata.creationTimestamp)' > state.json
ruby -ryaml -rjson -e 'puts YAML.dump(JSON.parse(STDIN.read))' < state.json > state.yaml
kops replace -f state.yaml --name $NAME
rm state.*

kops update cluster $NAME --yes

kubectl get nodes -o wide
```
## Cria uma nova AMI para o Kubernetes
```shell
aws ec2 deregister-image --image-id $K8S_BASE_AMI

cd packer/k8s-centos-base-new-kernel/
packer validate -var aws_base_ami=$CENTOS_BASE_AMI packer-template.json
packer build -var aws_base_ami=$CENTOS_BASE_AMI packer-template.json

export K8S_BASE_AMI=$(aws ec2 describe-images --owners self --filters Name=tag-key,Values=type Name=tag-value,Values=kubernetes --query 'Images[0].ImageId' --output text)
```
## Faz o rolling update de um instance group via Kops para usar a nova imagem gerada
```shell
cd ../..

kops get ig master-${AWS_DEFAULT_REGION}a -o yaml > state.yaml
ruby -ryaml -rjson -e 'puts YAML.load($stdin.read).to_json' < state.yaml | jq --arg IMAGE_ID $K8S_BASE_AMI '.spec.image |= $IMAGE_ID | del(.metadata.creationTimestamp)' > state.json
ruby -ryaml -rjson -e 'puts YAML.dump(JSON.parse(STDIN.read))' < state.json > state.yaml
kops replace -f state.yaml --name $NAME
rm state.*

kops update cluster --yes
kops rolling-update cluster
kops rolling-update cluster --yes

kubectl get nodes -o wide
```
