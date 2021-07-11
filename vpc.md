aws eks --region ap-northeast-2 update-kubeconfig --name btiwoom



1. aws eks describe-cluster --name btiwoom-svc --region ap-northeast-2 --query "cluster.identity.oidc.issuer"  
   --> "https://oidc.eks.ap-northeast-2.amazonaws.com/id/CEBD9B5CD4E751B3AA4E7676385B6D75"
2. eksctl utils associate-iam-oidc-provider --cluster btiwoom-svc --approve
   1. 확인방법
      aws iam list-open-id-connect-providers | grep CEBD9B5CD4E751B3AA4E7676385B6D75
   2. 결과 예 : "Arn": "arn:aws:iam::428923833419:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/CEBD9B5CD4E751B3AA4E7676385B6D75"

iampolicy arn
arn:aws:iam::428923833419:policy/AWSLoadBalancerControllerIAMPolicy



eksctl create iamserviceaccount \
  --cluster=btiwoom-svc \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::428923833419:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve        

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerAdditionalIAMPolicy \
  --policy-document file://iam_policy_v1_to_v2_additional.json

arn:aws:iam::428923833419:policy/AWSLoadBalancerControllerAdditionalIAMPolicy

arn:aws:iam::428923833419:role/eksctl-btiwoom-svc-addon-iamserviceaccount-k-Role1-1P1DNU4TV5AW8

aws iam attach-role-policy \
  --role-name eksctl-btiwoom-svc-addon-iamserviceaccount-k-Role1-1P1DNU4TV5AW8 \
  --policy-arn arn:aws:iam::428923833419:policy/AWSLoadBalancerControllerAdditionalIAMPolicy


# update k8s config

aws eks --region <region-code> update-kubeconfig --name <cluster_name>



ACCOUNT_ID=$(aws sts get-caller-identity --output text --query 'Account')

kubectl apply \
    --validate=false \
    -f https://github.com/jetstack/cert-manager/releases/download/v1.1.1/cert-manager.yaml



create sa

ADMIN=admin
kubectl create serviceaccount $ADMIN -n kube-system

cat << EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: $ADMIN
  namespace: kube-system
EOF

TOKEN=$(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep $ADMIN-token | awk '{print $1; exit}') -o jsonpath='{.data.token}' | base64 -d)
kubectl config set-credentials $ADMIN --token=$TOKEN