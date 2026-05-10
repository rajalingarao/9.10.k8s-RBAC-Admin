* Admin will create a Role for Harish, attach that Role to Harish using RoleBinding. He can be authenticated to AWS using AWS IAM aws configure on his project team like expense. Admin will give cluster access to Harish with Cluster, ClusterBinding.

Admin will create a Role for Harish and bind it using a RoleBinding. Harish will authenticate via AWS IAM using aws configure for the expense project. Admin will grant cluster access through a ClusterRole and ClusterRoleBinding.

For AWS authencation, We use AWS IAM.
For EKS authentication, We use same AWS IAM
for Authorization(getting specific resource access), We use Role,RoleBinding, Cluster, ClusterBinding.

* What is the difference between Role, RoleBinding, ClusterRole, and ClusterRoleBinding in terms of namespace?
Role: Namespace-level resource (namespace = true)
RoleBinding: Namespace-level resource (namespace = true)
ClusterRole: Cluster-level resource (namespace = false)
ClusterRoleBinding: Cluster-level resource (namespace = false)


* What are the steps to create and use RBAC resources in Kubernetes?
  * We will follow structure for project.

Note: These are admin or root user activities.

We will provide cluster describe access to Harish by create a policy IN IAM

IAM--> Policies-->Create Policy -->service:eks --> Accesss Level -->Read--> Describe
 Cluster

--> Specific Cluster 

Resource region: us-east-1
Resource Cluster name:expense-1
Resource ARN: automcaticay selected for expense-1

Policy Name: ExpenseEKSClusterDescribe

Then Go to Users--> create User:harish --> attach policies directly-->ExpenseEKSClusterDescribe--> add.

Now We create namespace called expense


First create namespace expense.

* k8-admin\namespace\expense\expense.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: expense
  labels:
    name: expense
    environment: dev
    
* k8-admin\namespace\expense\rbac.yaml
First we will give read access to pod

creating a Role called expense-pod-reader i.e Role
attaching this role to someone like harish   i.e RoleBinding

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: expense
  name: expense-pod-reader-role
rules:
- apiGroups: [""] #" indicates the core API group"
  resources: ["pods"]
  verbs: ["get","watch","list"]

---
apiVersion: rbac.authorization.k8s.io/v1
#This role binding allows "jane" to read pods in the "default" namespace.
#You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: expense-pod-reader-rolebinding
  namespace: expense
subjects:
* You can specify more than one "subject"
- kind: User
  name: harish #"name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
 * 'roleRef' specifies the binding to a Role/ClusterRole
 kind: Role #this must be Role or ClusterRole
 name: expense-pod-reader-role #this must match the name of the Role or ClusterRole you wish
 apiGroup: rbac.authorization.k8s.io

Note: How EKS knows to check AWS for Role and RoleBinding Authorization.
EKS should connect to AWS IAM using aws-auth.yaml configmap.



First we need to Authentication for EKS.
https://docs.aws.amazon.com/eks/latest/userguide/auth-configmap.html

* k8-admin\namespace\expense\aws-auth.yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::730335449147:role/eksctl-expense-1-nodegroup-spot-NodeInstanceRole-x53FJRVzJ6KD
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - groups:
      - expense-pod-reader-role
      userarn: arn:aws:iam::730335449147:user/harish
      username: harish
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system

kubectl get ns -n kube-system

kubectl get configmap aws-auth -n kube-system
kubectl get configmap aws-auth -n kube-system -o yaml

kind: ConfigMap
metadata:
  creationTimestamp: "2025-01-25T14:14:00Z"
  name: aws-auth
  namespace: kube-system
  uid: 34e2effb-2213-4446-b35d-cd9a4ab383b0



apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::730335449147:role/eksctl-expense-1-nodegroup-spot-NodeInstanceRole-x53FJRVzJ6KD
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - groups:
      - expense-pod-reader-role
      userarn: arn:aws:iam::730335449147:user/harish
      username: harish
kind: ConfigMap
metadata:
  creationTimestamp: "2025-01-25T14:14:00Z"
  name: aws-auth
  namespace: kube-system
  uid: 34e2effb-2213-4446-b35d-cd9a4ab383b0

To replace specific user arn, go to AWS IAM --> users-->harish --> get harish arn value

Now, Admin will email project team that namespace and access is given. Then

The project team has their own server like expense servers like creating ec2 instance.
Login into ec2-user  as harish user.

Harish wants to login AWS using 'aws configure'
Access Key:
Secret Access Key:
Region:

He can check his login status:
$aws sts get-caller-identity


he has cluster describe access, so he can get kubeconfig file access.
$aws eks update-kubeconfig --region us-east-1 --name expense

kubectl get nodes

We didn't install kubectl in nginx server of harish. We will install now.

need to install kubectl here.
---------------------------------------
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.0/2024-05-12/bin/linux/amd64/kubectl 
sudo chmod +x ./kubectl 
sudo mv kubectl  /usr/local/bin/ 
kubectl version --client 

$kubect get nodes

Error from server (Forbidden): nodes is forbidden: User "harish" cannot list resource "nodes" in API group "" at the cluster scope.

Note: if we observe above error, Authentication worked here, need to work on authorization by giving role access to him

$kubectl get pods -n expense
No resources found in expense namespace.

We have given access to harish only on expense namespace, not on default namespace.

Upto now Harish got access namespace level to access pods.

Now We will give cluster access to Harish by creating cluster and assigning using ClusterBinding.

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
# 'namespace' omitted since ClusterRole are not namespaced
  name: expense-cluster-reader-role
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  #objects in "secrets"
  resources: ["secrets","persistentvolumes","nodes"]
  verbs: ["get","watch","list"]

---
apiVersion: rbac.authorization.k8s.io/v1
#This cluster role binding allows anyone in the "manager" group to read secrets in any namespace
kind: ClusterRoleBinding
metadata:
  name: expense-cluster-reader-rolebinding
subjects:
- kind: User
  name: harish #Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: expense-cluster-reader-role
  apiGroup: rbac.authorization.k8s.io


$kubectl get nodes
$kubectl get secrets
$kubectl get pv
$kubectl get pvc   --> this is in default namespace, so not accessible.
kubectl get pvc
Error from server (Forbidden): persistentvolumeclaims is forbidden: User "harish" cannot list resource "persistentvolumeclaims" in API group "" in the namespace "default"

Note: Harish can have access only expense cluster, but didn't have the default cluster access.

namespace is true, resource is namespace level
namespace is false, then resource is cluster level.

1. harish has access to eks describe cluster or not
2. get aws-auth configmap
3. user harish authenticated
4. check role and role bindings

Note: In a cluster, there are multiple namespaces. if a user gets one cluster access, then he can able to access all its namespaces.

# Important Point:
* Question: If a user has cluster-level access in Kubernetes, can they access all namespaces such as `default` and `expense`? Also, does Kubernetes have a default cluster or only a default namespace?   

If you have cluster-level RBAC access (like cluster-admin), you can access resources across all namespaces such as default, expense, etc.

Kubernetes does not have a “default cluster.”
It has a built-in default namespace, which is used when no namespace is specified.


