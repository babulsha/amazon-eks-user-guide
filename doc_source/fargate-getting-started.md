# Getting Started with AWS Fargate on Amazon EKS<a name="fargate-getting-started"></a>

This topic helps you to get started running pods on AWS Fargate with your Amazon EKS cluster\.

**Note**  
AWS Fargate with Amazon EKS is currently only available in the following Regions:  


| Region Name | Region | 
| --- | --- | 
| US East \(Ohio\) | us\-east\-2 | 
| US East \(N\. Virginia\) | us\-east\-1 | 
| Asia Pacific \(Tokyo\) | ap\-northeast\-1 | 
| EU \(Ireland\) | eu\-west\-1 | 

If you restrict access to your cluster's public endpoint using CIDR blocks, it is recommended that you also enable private endpoint access so that Fargate pods can communicate with the cluster\. Without the private endpoint enabled, the CIDR blocks that you specify for public access must include the egress sources from your VPC\. For more information, see [Amazon EKS Cluster Endpoint Access Control](cluster-endpoint.md)\. 

## \(Optional\) Create a Cluster<a name="fargate-gs-create-cluster"></a>

Pods running on Fargate are supported on Amazon EKS clusters beginning with Kubernetes version 1\.14 and [platform version](platform-versions.md) `eks.5`\. Existing clusters can update to version 1\.14 to take advantage of this feature\. For more information, see [Updating an Amazon EKS Cluster Kubernetes Version](update-cluster.md)\. Existing 1\.14 clusters will be automatically updated to `eks.5` over time to support this feature\.

If you do not already have an Amazon EKS cluster that supports Fargate, you can create one with the following `eksctl` command\.

**Note**  
This procedure assumes that you have installed `eksctl`, and that your `eksctl` version is at least `0.11.1`\. You can check your version with the following command:  

```
eksctl version
```
 For more information on installing or upgrading `eksctl`, see [Installing or Upgrading `eksctl`](eksctl.md#installing-eksctl)\.

```
eksctl create cluster --name my-cluster --version 1.14 --fargate
```

Adding the `--fargate` option in the command above creates a cluster without a node group\. However, `eksctl` creates a pod execution role, a Fargate profile for the `default` and `kube-system` namespaces, and it patches the `coredns` deployment so that it can run on Fargate\. 

## Ensure that Existing Nodes can Communicate with Fargate Pods<a name="fargate-gs-check-compatibility"></a>

If you are working with a new cluster with no worker nodes, or a cluster with only [managed node groups](managed-node-groups.md), you can skip to [Create a Fargate Pod Execution Role](#fargate-sg-pod-execution-role)\.

If you are working with an existing cluster that already has worker nodes associated with it, you need to make sure that pods on these nodes can communicate freely with pods running on Fargate\. Pods running on Fargate are automatically configured to use the cluster security group for the cluster that they are associated with\. You must ensure that any existing worker nodes in your cluster can send and receive traffic to and from the cluster security group\. [Managed Node Groups](managed-node-groups.md) are automatically configured to use the cluster security group as well, so you do not need to modify or check them for this compatibility\.

For existing node groups that were created with `eksctl` or the Amazon EKS\-managed AWS CloudFormation templates, you can add the cluster security group to the nodes manually, or you can modify the node group's Auto Scaling group launch template to attach the cluster security group to the instances\. For more information, see [Changing an Instance's Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html#SG_Changing_Group_Membership) in the *Amazon VPC User Guide*\.

You can check for a cluster security group for your cluster in the AWS Management Console under the cluster's **Networking** section, or with the following AWS CLI command:

```
aws eks describe-cluster --name cluster_name --query cluster.resourcesVpcConfig.clusterSecurityGroupId
```

## Create a Fargate Pod Execution Role<a name="fargate-sg-pod-execution-role"></a>

When your cluster creates pods on AWS Fargate, the pods need to make calls to AWS APIs on your behalf to do things like pull container images from Amazon ECR\. The Amazon EKS pod execution role provides the IAM permissions to do this\.

**Note**  
If you created your cluster with `eksctl` using the `--fargate` option, then your cluster already has a pod execution role and you can skip ahead to [Create a Fargate Profile for your Cluster](#fargate-gs-create-profile)\. Similarly, if you use `eksctl` to create your Fargate profiles, `eksctl` will create your pod execution role if one does not already exist\.

When you create a Fargate profile, you must specify a pod execution role to use with your pods\. This role is added to the cluster's Kubernetes [Role Based Access Control](https://kubernetes.io/docs/admin/authorization/rbac/) \(RBAC\) for authorization\. This allows the `kubelet` that is running on the Fargate infrastructure to register with your Amazon EKS cluster so that it can appear in your cluster as a node\. For more information, see [Pod Execution Role](pod-execution-role.md)\.

1. Open the IAM console at [https://console\.aws\.amazon\.com/iam/](https://console.aws.amazon.com/iam/)\.

1. Choose **Roles**, then **Create role**\.

1. Choose **EKS** from the list of services, **EKS \- Fargate pod** for your use case, and then **Next: Permissions**\.

1. Choose **Next: Tags**\.

1. \(Optional\) Add metadata to the role by attaching tags as key–value pairs\. For more information about using tags in IAM, see [Tagging IAM Entities](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_tags.html) in the *IAM User Guide*\. 

1. Choose **Next: Review**\.

1. For **Role name**, enter a unique name for your role, such as `AmazonEKSFargatePodExecutionRole`, then choose **Create role**\.

## Create a Fargate Profile for your Cluster<a name="fargate-gs-create-profile"></a>

Before you can schedule pods running on Fargate in your cluster, you must define a Fargate profile that specifies which pods should use Fargate when they are launched\. For more information, see [AWS Fargate Profile](fargate-profile.md)\.

**Note**  
If you created your cluster with `eksctl` using the `--fargate` option, then a Fargate profile has already been created for your cluster with selectors for all pods in the `kube-system` and `default` namespaces\. Use the following procedure to create Fargate profiles for any other namespaces you would like to use with Fargate\.

Choose the tab below that corresponds to your preferred Fargate profile creation method\.

------
#### [ eksctl ]

**To create a Fargate profile for a cluster with `eksctl`**

This procedure assumes that you have installed `eksctl`, and that your `eksctl` version is at least `0.11.1`\. You can check your version with the following command:

```
eksctl version
```

 For more information on installing or upgrading `eksctl`, see [Installing or Upgrading `eksctl`](eksctl.md#installing-eksctl)\.
+ Create your Fargate profile with the following `eksctl` command, replacing the *variable text* with your own values\. You must specify a namespace, but the labels option is not required\.

  ```
  eksctl create fargateprofile --cluster cluster_name --name fargate_profile_name --namespace kubernetes_namespace --labels key=value
  ```

------
#### [ AWS Management Console ]

**To create a Fargate profile for a cluster with the AWS Management Console**

1. Open the Amazon EKS console at [https://console\.aws\.amazon\.com/eks/home\#/clusters](https://console.aws.amazon.com/eks/home#/clusters)\.

1. Choose the cluster to create a Fargate profile for\.

1. Under **Fargate profiles**, choose **Add Fargate profile**\.

1. On the **Configure Fargate profile** page, enter the following information and choose **Next**\.

   1. For **Name**, enter a unique name for your Fargate profile\.

   1. For **Pod execution role**, choose the pod execution role to use with your Fargate profile\. Only IAM roles with the `eks-fargate-pods.amazonaws.com` service principal are shown\. If you do not see any roles listed here, you must create one\. For more information, see [Pod Execution Role](pod-execution-role.md)\.

   1. For **Subnets**, choose the subnets to use for your pods\. By default, all subnets in your cluster's VPC are selected\. Only private subnets are supported for pods running on Fargate; you must deselect any public subnets\.

   1. For **Tags**, you can optionally tag your Fargate profile\. These tags do not propagate to other resources associated with the profile, such as its pods\.

1. On the **Configure pods selection** page, enter the following information and choose **Next**\.

   1. For **Namespace**, enter a namespace to match for pods, such as `kube-system` or `default`\.

   1. \(Optional\) Add Kubernetes labels to the selector that pods in the specified namespace must have to match the selector\. For example, you could add the label `infrastructure: fargate` to the selector so that only pods in the specified namespace that also have the `infrastructure: fargate` Kubernetes label match the selector\.

1. On the **Review and create** page, review the information for your Fargate profile and choose **Create**\.

------

## \(Optional\) Update CoreDNS<a name="fargate-gs-coredns"></a>

By default, CoreDNS is configured to run on Amazon EC2 infrastructure on Amazon EKS clusters\. If you want to *only* run your pods on Fargate in your cluster, you need to modify the CoreDNS deployment to remove the `eks.amazonaws.com/compute-type : ec2` annotation\. You would also need to create a Fargate profile to target the CoreDNS pods\. The following Fargate profile JSON file does this\.

**Note**  
If you created your cluster with `eksctl` using the `--fargate` option, then `coredns` has already been patched to run on Fargate and you can skip ahead to [Next Steps](#fargate-gs-next-steps)\.

```
{
    "fargateProfileName": "coredns",
    "clusterName": "dev",
    "podExecutionRoleArn": "arn:aws:iam::111122223333:role/AmazonEKSFargatePodExecutionRole",
    "subnets": [
        "subnet-0b64dd020cdff3864",
        "subnet-00b03756df55e2b87",
        "subnet-0418fcb68ed294abf"
    ],
    "selectors": [
        {
            "namespace": "kube-system",
            "labels": {
                "k8s-app": "kube-dns"
            }
        }
    ]
}
```

You could apply this Fargate profile to your cluster with the following AWS CLI command\. First, create a file called `coredns.json` and paste the JSON file from the previous step into it, replacing the *variable text* with your own cluster values\.

```
aws eks create-fargate-profile --cli-input-json file://coredns.json
```

Then, use the following `kubectl` command to remove the `eks.amazonaws.com/compute-type : ec2` annotation from the CoreDNS pods\.

```
kubectl patch deployment coredns -n kube-system --type json \
-p='[{"op": "remove", "path": "/spec/template/metadata/annotations/eks.amazonaws.com~1compute-type"}]'
```

## Next Steps<a name="fargate-gs-next-steps"></a>
+ You can start migrating your existing applications to run on Fargate with the following workflow\.

  1. [Create a Fargate profile](fargate-profile.md#create-fargate-profile) that matches your application's Kubernetes namespace and Kubernetes labels\.

  1. Delete and re\-create any existing pods so that they are scheduled on Fargate\. For example, the following command triggers a rollout of the `coredns` Deployment\. You can modify the namespace and deployment type to update your specific pods\.

     ```
     kubectl rollout restart -n kube-system deployment coredns
     ```
+ Deploy the [ALB Ingress Controller on Amazon EKS](alb-ingress.md) \(version v1\.1\.4 or later\) to allow Ingress objects for your pods running on Fargate\.
+ Deploy the [Vertical Pod Autoscaler](vertical-pod-autoscaler.md) for your pods running on Fargate to optimize the CPU and memory used for your applications\. Be sure to set the pod update policy to either `Auto` or `Recreate` to ensure correct functionality\.