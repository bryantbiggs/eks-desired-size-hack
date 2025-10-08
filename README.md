# EKS Desired Size Hack

## Problem

Within Terraform, there exists a situation where a resource is provisioned that scales to multiple copies of a resource, but the management of that scaling is not intended to be controlled by Terraform. Some examples include:

- EKS clusters: scaling EKS managed node groups or self-managed node groups
- AWS Auto Scaling groups (i.e. - EKS self-managed node groups)
- ECS clusters: scaling the number of ECS tasks
- Aurora RDS: scaling the number of read replicas

In each of these scenarios, it is very common for something other than Terraform to control the desired quantity of resources which then poses a conflict. Terraform assumes full control over the resources it manages, those defined within its statefile, and if not considered, the desired number of resources can be forcibly changed back to the quantity known by Terraform on the next `terraform apply`. Therefore, it has become common practice to ignore these changes by prescribing a `lifecycle` block within the resource definition, adding the attribute(s) that are to be controlled outside of Terraform to the list of [`ignore_changes`](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle#ignore_changes).

For EKS clusters created with the https://github.com/terraform-aws-modules/terraform-aws-eks Terraform module, the `desired_size` of both the EKS and self-managed node groups created by the module are ignored by default for this reason. It is assumed that most users will utilize a form of autoscaling that will manage the `desired_size` attribute of the node groups (i.e. - Cluster Autoscaler will manage this attribute for scaling), and therefore Terraform should not attempt to manage after it has been initially set during creation of the node group.

- [EKS managed node group reference](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/ca03fd9ec16f8e6d8440d23adb72cb7c7ba33c56/modules/eks-managed-node-group/main.tf#L383)
- [Self-managed node group reference](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/ca03fd9ec16f8e6d8440d23adb72cb7c7ba33c56/modules/self-managed-node-group/main.tf#L673)

However, in the case of the EKS module, this can create a situation where users are unable to over-provision, or pre-provision node resources by raising the `min_size` above the current `desired_size` setting. To make this scenario a bit more clear, consider the following example:

- A cluster is currently configured with `min_size = 3`, `desired_size = 5`, and `max_size = 50`
- A promotional event is scheduled to take place where traffic will spike to 100x the normal traffic in a short period of time
- The user would like to pre-provision the cluster with `min_size = 40`, `desired_size = 50`, and `max_size = 100` to ensure that the cluster can handle the traffic spike

In this scenario, the user wants to force more nodes out into the cluster, even though there currently isn't any traffic that warrants the additional nodes. Because the `desired_size` is currently set to `5`, and this has been ignored from Terraform's control, the user is unable to raise the `min_size` above `5` without first raising the `desired_size` through some other means (i.e. - manually through the console, the awscli, etc.). If you attempt to raise the `min_size` to a value above the `desired_size`, the AWS API will return an error stating that the `min_size` cannot be greater than the `desired_size`.

In addition, Terraform currently does not support variables within the [`lifecycle` meta-argument block](https://github.com/hashicorp/terraform/issues/3116) which means that the `ignore_changes` are essentially hardcoded for all users.

## One Solution

One possible solution using Terraform is shown in `main.tf` using the `null_resource`:


```hcl
resource "null_resource" "update_desired_size" {
  triggers = {
    desired_size = local.desired_size
  }

  provisioner "local-exec" {
    interpreter = ["/bin/bash", "-c"]

    # Note: this requires the awscli to be installed locally where Terraform is executed
    command     = <<-EOT
      aws eks update-nodegroup-config \
        --cluster-name ${module.eks.cluster_name} \
        --nodegroup-name ${element(split(":", module.eks.eks_managed_node_groups["green"].node_group_id), 1)} \
        --scaling-config desiredSize=${local.desired_size}
    EOT
  }
}
```

### How This Works

Once the EKS module is provisioned, the node group(s) have their initial `desired_size` set by what is prescribed in the Terraform configuration for the cluster. To update the `desired_size` of the node group(s), the user would update the local variable for `desired_size` which will cause the `null_resource` trigger to re-execute the commands provided in the `local-exec` provisioner. Within the `local-exec`, we are making an EKS API call to update the scaling config of the node group provided to set the `desiredSize` to the value of the local varaible `desired_size`. This will only get executed once for each time the `local.desired_size` value is changed, and will not be executed on subsequent `terraform apply` runs.

Note: the `desired_size` must be raised BEFORE the `min_size` can be raised to a value greater than the `desired_size`. This will typically be in the form of two subsequent, but separate, `terraform apply` steps. One to raise the `desired_size` followed by one to raise the `min_size`.

## References

- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/34
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/38
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/143
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/162
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/291
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/510
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/681
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/748
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/835
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/836
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/944
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1222
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1295
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1568
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1879
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1924
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2030
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2151
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2158
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/2236
- https://github.com/terraform-aws-modules/terraform-aws-eks/issues/3521
