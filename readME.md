# Ingress Controller
# ULTIMATE ROLE IS TO INSTALL ALB CONTROLLER IN KUBERNETES
---------------------------------------------------------------------
1. OIDC provider: --> to use IAM roles we need this provider
Need of OIDC provider
Let's say you have an application running in a Kubernetes Pod that needs to read/write data to S3.

With OIDC Provider: ✅ You create an IAM role with S3 permissions and link it to a Kubernetes service account.
✅ Only Pods using that service account can access S3.

Without OIDC Provider: ❌ The IAM role can exist, but your Pod can’t assume it because AWS doesn’t trust Kubernetes identities without an OIDC provider.
❌ The only workaround is granting permissions to the entire EC2 node, which means all Pods running on that node get S3 access (even if they don’t need it).

```
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster expense-dev \
    --approve
```
---------------------------------------------------------------------
2. IAM policy content json download for aws alb controller 

```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.10.0/docs/install/iam_policy.json
```
---------------------------------------------------------------------
3. creating a policy using the above content
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```
---------------------------------------------------------------------
4. creating a service account ,  attaching policy to IAM role(role will get created by k8) and attaching that role  to k8 account 

Whats the below command doing ?
Since Kubernetes does not natively understand AWS IAM, we manually create a Service Account and link it to an IAM Role using IAM Roles for Service Accounts (IRSA). This lets Kubernetes pods securely assume AWS IAM roles.

eksctl creates an IAM role behind the scenes automatically.
It attaches the specified policy (AWSLoadBalancerControllerIAMPolicy) to that role.
Then, it associates the role with the Kubernetes service account (aws-load-balancer-controller).

Who Gets the IAM Role?
The IAM role is not attached to your IAM user. 🚫

It is linked to the Kubernetes service account (aws-load-balancer-controller) inside the kube-system namespace. ✅

Any Pod using the service account (aws-load-balancer-controller) in EKS can assume this role.

```
eksctl create iamserviceaccount \
--cluster=expense-dev \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::180294178330:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region us-east-1 \
--approve
```
---------------------------------------------------------------------
5. creating a helm chart
```
helm repo add eks https://aws.github.io/eks-charts
```
---------------------------------------------------------------------
6. installing alb controller to controll aws alb in that repo  
Whats the below command doing ?

This command installs the AWS Load Balancer Controller in your EKS cluster using Helm.

✅ The AWS Load Balancer Controller gets deployed in EKS.

✅ It uses the IAM service account (aws-load-balancer-controller) to get the required permissions.

✅ Now, Kubernetes Ingress and Service objects can automatically create ALBs and NLBs

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=expense-dev --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```
---------------------------------------------------------------------
