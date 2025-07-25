# User Data & Bootstrapping

Users can see the various methods of using and providing user data through the [user data tests](https://github.com/terraform-aws-modules/terraform-aws-eks/tree/master/tests/user-data) as well more detailed information on the design and possible configurations via the [user data module itself](https://github.com/terraform-aws-modules/terraform-aws-eks/tree/master/modules/_user_data)

## Summary

- AWS EKS Managed Node Groups
  - By default, any supplied user data is pre-pended to the user data supplied by the EKS Managed Node Group service
  - If users supply an `ami_id`, the service no longers supplies user data to bootstrap nodes; users can enable `enable_bootstrap_user_data` and use the module provided user data template, or provide their own user data template
  - AMI types of `BOTTLEROCKET_*`, user data must be in TOML format
  - AMI types of `WINDOWS_*`, user data must be in powershell/PS1 script format
- Self Managed Node Groups
  - `AL2_*` AMI types -> the user data template (bash/shell script) provided by the module is used as the default; users are able to provide their own user data template
  - `AL2023_*` AMI types -> the user data template (MIME multipart format) provided by the module is used as the default; users are able to provide their own user data template
  - `BOTTLEROCKET_*` AMI types -> the user data template (TOML file) provided by the module is used as the default; users are able to provide their own user data template
  - `WINDOWS_*` AMI types -> the user data template (powershell/PS1 script) provided by the module is used as the default; users are able to provide their own user data template

The templates provided by the module can be found under the [templates directory](https://github.com/terraform-aws-modules/terraform-aws-eks/tree/master/templates)

## EKS Managed Node Group

When using an EKS managed node group, users have 2 primary routes for interacting with the bootstrap user data:

1. If a value for `ami_id` is not provided, users can supply additional user data that is pre-pended before the EKS Managed Node Group bootstrap user data. You can read more about this process from the [AWS supplied documentation](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html#launch-template-user-data)

   - Users can use the following variables to facilitate this process:

    For `AL2_*`, `BOTTLEROCKET_*`, and `WINDOWS_*`:
    ```hcl
    pre_bootstrap_user_data = "..."
    ```

    For `AL2023_*`
    ```hcl
    cloudinit_pre_nodeadm = [{
      content      = <<-EOT
        ---
        apiVersion: node.eks.aws/v1alpha1
        kind: NodeConfig
        spec:
          ...
      EOT
      content_type = "application/node.eks.aws"
    }]
    ```

2. If a custom AMI is used, then per the [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html#launch-template-custom-ami), users will need to supply the necessary user data to bootstrap and register nodes with the cluster when launched. There are two routes to facilitate this bootstrapping process:
   - If the AMI used is a derivative of the [AWS EKS Optimized AMI ](https://github.com/awslabs/amazon-eks-ami), users can opt in to using a template provided by the module that provides the minimum necessary configuration to bootstrap the node when launched:
     - Users can use the following variables to facilitate this process:
       ```hcl
       enable_bootstrap_user_data = true # to opt in to using the module supplied bootstrap user data template
       pre_bootstrap_user_data    = "..."
       bootstrap_extra_args       = "..."
       post_bootstrap_user_data   = "..."
       ```
   - If the AMI is **NOT** an AWS EKS Optimized AMI derivative, or if users wish to have more control over the user data that is supplied to the node when launched, users have the ability to supply their own user data template that will be rendered instead of the module supplied template. Note - only the variables that are supplied to the `templatefile()` for the respective AMI type are available for use in the supplied template, otherwise users will need to pre-render/pre-populate the template before supplying the final template to the module for rendering as user data.
     - Users can use the following variables to facilitate this process:
       ```hcl
       user_data_template_path  = "./your/user_data.sh" # user supplied bootstrap user data template
       pre_bootstrap_user_data  = "..."
       bootstrap_extra_args     = "..."
       post_bootstrap_user_data = "..."
       ```

| ℹ️ When using bottlerocket, the supplied user data (TOML format) is merged in with the values supplied by EKS. Therefore, `pre_bootstrap_user_data` and `post_bootstrap_user_data` are not valid since the bottlerocket OS handles when various settings are applied. If you wish to supply additional configuration settings when using bottlerocket, supply them via the `bootstrap_extra_args` variable. For the `AL2_*` AMI types, `bootstrap_extra_args` are settings that will be supplied to the [AWS EKS Optimized AMI bootstrap script](https://github.com/awslabs/amazon-eks-ami/blob/master/files/bootstrap.sh#L14) such as kubelet extra args, etc. See the [bottlerocket GitHub repository documentation](https://github.com/bottlerocket-os/bottlerocket#description-of-settings) for more details on what settings can be supplied via the `bootstrap_extra_args` variable. |
| :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

### Self Managed Node Group

Self managed node groups require users to provide the necessary bootstrap user data. Users can elect to use the user data template provided by the module for their respective AMI type or provide their own user data template for rendering by the module.

- If the AMI used is a derivative of the [AWS EKS Optimized AMI ](https://github.com/awslabs/amazon-eks-ami), users can opt in to using a template provided by the module that provides the minimum necessary configuration to bootstrap the node when launched:
  - Users can use the following variables to facilitate this process:
    ```hcl
    enable_bootstrap_user_data = true # to opt in to using the module supplied bootstrap user data template
    pre_bootstrap_user_data    = "..."
    bootstrap_extra_args       = "..."
    post_bootstrap_user_data   = "..."
    ```
  - If the AMI is **NOT** an AWS EKS Optimized AMI derivative, or if users wish to have more control over the user data that is supplied to the node when launched, users have the ability to supply their own user data template that will be rendered instead of the module supplied template. Note - only the variables that are supplied to the `templatefile()` for the respective AMI type are available for use in the supplied template, otherwise users will need to pre-render/pre-populate the template before supplying the final template to the module for rendering as user data.
    - Users can use the following variables to facilitate this process:
      ```hcl
      user_data_template_path  = "./your/user_data.sh" # user supplied bootstrap user data template
      pre_bootstrap_user_data  = "..."
      bootstrap_extra_args     = "..."
      post_bootstrap_user_data = "..."
      ```

### Logic Diagram

The rough flow of logic that is encapsulated within the `_user_data` module can be represented by the following diagram to better highlight the various manners in which user data can be populated.

<p align="center">
  <img src="https://raw.githubusercontent.com/terraform-aws-modules/terraform-aws-eks/master/.github/images/user_data.svg" alt="User Data" width="60%">
</p>
