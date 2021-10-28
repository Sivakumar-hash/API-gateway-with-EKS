# API-gateway-with-EKS

Launch t2.micro server from AWS console

prerequisite : 

#AWS CLI version 2
https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html

#eksctl
https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

#kubectl
https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

#Helm
https://helm.sh/docs/intro/install/


#Configure environment variable on Server
  export AGW_AWS_REGION=eu-west-1 <-- Change this to match your region
  export AGW_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
  export AGW_EKS_CLUSTER_NAME=eks-siva-cluster

#Create a new EKS cluster using eksctl:
  eksctl create cluster \
  --name $AGW_EKS_CLUSTER_NAME \
  --region $AGW_AWS_REGION \
  --managed
  
  
#Deploy the AWS Load Balancer Controller
  ## Associate OIDC provider 
  eksctl utils associate-iam-oidc-provider \
  --region $AGW_AWS_REGION \
  --cluster $AGW_EKS_CLUSTER_NAME \
  --approve
  ## Download the IAM policy document
  curl -S https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json -o iam-policy.json

  
 ## Create a service account 
  eksctl create iamserviceaccount \
  --cluster=$AGW_EKS_CLUSTER_NAME \
  --region $AGW_AWS_REGION \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --override-existing-serviceaccounts \
  --attach-policy-arn=arn:aws:iam::${AGW_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy-APIGWDEMO \
  --approve
  
  ## Get EKS cluster VPC ID
  export AGW_VPC_ID=$(aws eks describe-cluster \
  --name $AGW_EKS_CLUSTER_NAME \
  --region $AGW_AWS_REGION  \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)
  
  helm repo add eks https://aws.github.io/eks-charts && helm repo update
  kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
  helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=$AGW_EKS_CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$AGW_VPC_ID\
  --set region=$AGW_AWS_REGION
  
 #Deploy the ACK Controller for API Gateway
   Create a service account for ACK
   curl -O https://raw.githubusercontent.com/aws-samples/amazon-apigateway-ingress-controller-blog/Mainline/apigw-ingress-controller-blog/ack-iam-policy.json

  aws iam create-policy \
  --policy-name ACKIAMPolicy \
  --policy-document file://ack-iam-policy.json

  eksctl create iamserviceaccount \
  --attach-policy-arn=arn:aws:iam::${AGW_ACCOUNT_ID}:policy/ACKIAMPolicy \
  --cluster=$AGW_EKS_CLUSTER_NAME \
  --namespace=kube-system \
  --name=ack-apigatewayv2-controller \
  --override-existing-serviceaccounts \
  --region $AGW_AWS_REGION \
  --approve
 
  export HELM_EXPERIMENTAL_OCI=1
  export SERVICE=apigatewayv2
  export RELEASE_VERSION=v0.0.2
  export CHART_EXPORT_PATH=/tmp/chart
  export CHART_REPO=public.ecr.aws/aws-controllers-k8s/chart
  export CHART_REF=$CHART_REPO:$SERVICE-$RELEASE_VERSION
  mkdir -p $CHART_EXPORT_PATH

  helm chart pull $CHART_REF
  helm chart export $CHART_REF --destination $CHART_EXPORT_PATH
  
  # Install ACK
  helm install ack-${SERVICE}-controller \
  $CHART_EXPORT_PATH/ack-${SERVICE}-controller \
  --namespace kube-system \
  --set serviceAccount.create=false \
  --set aws.region=$AGW_AWS_REGION
  
    
#Deploy node js applications

kubectl apply -f https://github.com/Sivakumar-hash/nodejs-EKS/blob/master/deployment.yaml

kubectl apply -f https://github.com/Sivakumar-hash/nodejs-EKS/blob/master/service.yaml

#Creat API Gateway resources
AGW_VPCLINK_SG=$(aws ec2 create-security-group \
  --description "SG for VPC Link" \
  --group-name SG_VPC_LINK \
  --vpc-id $AGW_VPC_ID \
  --region $AGW_AWS_REGION \
  --output text \
  --query 'GroupId')
  
#Create a VPC Link for the internal NLB

cat > vpclink.yaml<<EOF
apiVersion: apigatewayv2.services.k8s.aws/v1alpha1
kind: VPCLink
metadata:
  name: nlb-internal
spec:
  name: nlb-internal
  securityGroupIDs: 
    - $AGW_VPCLINK_SG
  subnetIDs: 
    - $(aws ec2 describe-subnets \
          --filter Name=tag:kubernetes.io/role/internal-elb,Values=1 \
          --query 'Subnets[0].SubnetId' \
          --region $AGW_AWS_REGION --output text)
    - $(aws ec2 describe-subnets \
          --filter Name=tag:kubernetes.io/role/internal-elb,Values=1 \
          --query 'Subnets[1].SubnetId' \
          --region $AGW_AWS_REGION --output text)
    - $(aws ec2 describe-subnets \
          --filter Name=tag:kubernetes.io/role/internal-elb,Values=1 \
          --query 'Subnets[2].SubnetId' \
          --region $AGW_AWS_REGION --output text)
          EOF
       kubectl apply -f vpclink.yaml
                        
Create an API with VPC Link integration:
  cat > apigw-api.yaml<<EOF
apiVersion: apigatewayv2.services.k8s.aws/v1alpha1
kind: API
metadata:
  name: apitest-private-nlb
spec:
  body: '{
              "openapi": "3.0.1",
              "info": {
                "title": "ack-apigwv2-import-test-private-nlb",
                "version": "v1"
              },
              "paths": {
              "/\$default": {
                "x-amazon-apigateway-any-method" : {
                "isDefaultRoute" : true,
                "x-amazon-apigateway-integration" : {
                "payloadFormatVersion" : "1.0",
                "connectionId" : "$(kubectl get vpclinks.apigatewayv2.services.k8s.aws \
  nlb-internal \
  -o jsonpath="{.status.vpcLinkID}")",
      "type" : "http_proxy",
                "httpMethod" : "GET",
                "uri" : "$(aws elbv2 describe-listeners \
  --load-balancer-arn $(aws elbv2 describe-load-balancers \
  --region $AGW_AWS_REGION \
  --query "LoadBalancers[?contains(DNSName, '$(kubectl get service authorservice \
  -o jsonpath="{.status.loadBalancer.ingress[].hostname}")')].LoadBalancerArn" \
  --output text) \
  --region $AGW_AWS_REGION \
  --query "Listeners[0].ListenerArn" \
  --output text)",
               "connectionType" : "VPC_LINK"
                  }
                }
              },
              "/meta": {
                  "get": {
                    "x-amazon-apigateway-integration": {
                       "uri" : "$(aws elbv2 describe-listeners \
  --load-balancer-arn $(aws elbv2 describe-load-balancers \
  --region $AGW_AWS_REGION \
  --query "LoadBalancers[?contains(DNSName, '$(kubectl get service echoserver \
  -o jsonpath="{.status.loadBalancer.ingress[].hostname}")')].LoadBalancerArn" \
  --output text) \
  "httpMethod": "GET",
                      "connectionId": "$(kubectl get vpclinks.apigatewayv2.services.k8s.aws \
  nlb-internal \
  -o jsonpath="{.status.vpcLinkID}")",
                      "type": "HTTP_PROXY",
                      "connectionType": "VPC_LINK",
                      "payloadFormatVersion": "1.0"
                    }
                  }
                }
              },
              "components": {}
        }'
EOF
  kubectl apply -f apigw-api.yaml
  
  
#Create a stage
 echo "
apiVersion: apigatewayv2.services.k8s.aws/v1alpha1
kind: Stage
metadata:
  name: "apiv1"
spec:
  apiID: $(kubectl get apis.apigatewayv2.services.k8s.aws apitest-private-nlb -o=jsonpath='{.status.apiID}')
  stageName: api
  autoDeploy: true
" | kubectl apply -f -
  
#Invoke API
    curl $(kubectl get api apitest-private-nlb -o jsonpath="{.status.apiEndpoint}")/users/123

