# Deploying a Databricks AWS E2 Workspace using a Cloud Formation Template

__This repo contains an AWS sample cloud formation template for deploying a Databricks workspace on AWS using the E2 deployment architecture.__

---
## Description

A Databricks workspace is an environment for accessing all of your Databricks assets. The workspace organizes objects (notebooks, libraries, and experiments) into folders, and provides access to data and computational resources such as clusters and jobs.

The Databricks workspace is broken down into a control plane and a data plane:  

  * The control plane includes the backend services that Databricks manages in its own AWS account. Notebook commands and many other workspace configurations are stored in the control plane and encrypted at rest.
  * The data plane is where your data is processed.  The data plane resides within the customer's AWS account.  All compute and storage reside within the data plane.

AWS CloudFormation provides an "infrastructure-as-code" approach for deploying AWS services.  This CloudFormation repo allows Databricks customers to deploy all of the necessary workspace infrastructure on AWS through a template.  The template offers the following security options available:

  * [Customer-managed VPCs](https://docs.databricks.com/administration-guide/cloud-configurations/aws/customer-managed-vpc.html): Create Databricks workspaces in your own VPC rather than using the default architecture in which clusters are created in a single AWS VPC that Databricks creates and configures in your AWS account.
  * [Secure cluster connectivity](https://docs.databricks.com/security/secure-cluster-connectivity.html): Also known as “No Public IPs,” secure cluster connectivity lets you launch clusters in which all nodes have only private IP addresses, providing enhanced security.
  * [Customer-managed keys for managed services](https://docs.databricks.com/security/keys/customer-managed-keys-managed-services-aws.html): Provide KMS keys to encrypt notebook and secret data in the Databricks-managed control plane.
  * [PrivateLink](https://docs.databricks.com/administration-guide/cloud-configurations/aws/privatelink.html): Use AWS PrivateLink to enable private connectivity between clusters on the data plane and core services on the control plane within the Databricks workspace infrastructure.

## Prerequisites

  * [Databricks Account](https://accounts.cloud.databricks.com/login):  In order to use this template, customers must create an account and elect AWS as the cloud provider.
  * Administrator access to an AWS account
  * Fully configured access to the AWS account through the console or the AWS CLI.  The following steps assume AWS CLI usage.

## Installation

  * Copy the *_params.json* file.  Rename the clone appropriately (i.e. params.json).
  * Here is a description of the available configuration parameters:
    * __AccountId__: The Account ID of your [Databricks AWS account](https://accounts.cloud.databricks.com).  The Account ID can be retrieved through the [Databricks Account portal](https://accounts.cloud.databricks.com).
    * __Username__: Account email for authenticating the REST API. Note that this value is case sensitive.  This value must be a valid email format.  
    * __Password__: Account password for authenticating the REST API. The minimum length is 8 alphanumeric characters.
    * __PricingTier__: Pricing tier of Databricks account, either STANDARD, PREMIUM, or ENTERPRISE.
    * __DeploymentName__: Choose this value carefully. The deployment name defines part of the workspace subdomain (e.g., workspace-deployment-name.cloud.databricks.com). This value must be unique across all deployments in all AWS Regions. It cannot start or end with a hyphen. If your account has a deployment-name prefix, add the prefix followed by a hyphen. For more information, see https://docs.databricks.com/administration-guide/account-api/new-workspace.html#step-5-create-the-workspace.
    * __AWSRegion__: AWS Region where the workspace is created. See the [supported regions](https://docs.databricks.com/administration-guide/cloud-configurations/aws/regions.html) and note the features available within each region.
    * __HIPAAparm__: Entering "Yes" creates a template for creating clusters in the HIPAA account.
    * __TagValue__: All new AWS objects get a tag with the key name. Enter a value to identify all new AWS objects that this template creates. For more information, see https://docs.aws.amazon.com/general/latest/gr/aws_tagging.html.
    * __ExistingCrossAccountIAMRoleARN__: An IAM cross-account role is required for the control plane to launch compute in the data plane.  If you already configured a cross-account IAM role adhering to [these requirements](https://docs.databricks.com/administration-guide/account-api/iam-role.html), specify the ARN of that IAM role here.  __IMPORTANT__: If you do provide a pre-configured cross-account IAM role, it must adhere to the [requirements](https://docs.databricks.com/administration-guide/account-api/iam-role.html) with all the appropriate policies.  Do not specify this value if you are looking to create the cross-account role through this template.
    * __ExistingCrossAccountIAMRoleName__: If an IAM cross-account role is configured already, the role name should be passed in this parameter.
    * __NewCrossAccountIAMRoleName__: Enter a unique cross-account IAM role name if you wish to have Databricks create the cross-account IAM role.  The IAM role will be configured according to [the Databricks cross-account IAM role documentation](https://docs.databricks.com/administration-guide/account-api/iam-role.html).
    * __BucketName__: The name of your root bucket.  This bucket will contain root storage for workspace objects like cluster logs, notebook revisions, and job results libraries.

    The following parameters only apply for a VPC that has already been configured:

    * __VPCID__: If you elect to configure your own VPC outside of this template, provide the VPC ID here.  The configured VPC must adhere to [these requirements](https://docs.databricks.com/administration-guide/cloud-configurations/aws/customer-managed-vpc.html).  __IMPORTANT__: If your VPC does not adhere to the [requirements](https://docs.databricks.com/administration-guide/cloud-configurations/aws/customer-managed-vpc.html), then this deployment will fail.
    * __SecurityGroupIDs__: Name of one or more VPC security groups. Only enter a value if you set VPCID. The format is sg-xxxxxxxxxxxxxxxxx. Use commas to separate multiple IDs. Databricks must have access to at least one security group but no more than five. You can reuse existing security groups. For more information, see https://docs.databricks.com/administration-guide/cloud-configurations/aws/customer-managed-vpc.html.
    * __SubnetIDs__: Enter at least two private subnet IDs [i.e. subnet-xxxx,subnet-yyyy]. Only enter a value if you set VPCID. Subnets cannot be shared with other workspaces or non-Databricks resources. Each subnet must be private, have outbound access, and a netmask between /17 and /25. The NAT gateway must have its own subnet that routes 0.0.0.0/0 traffic to an internet gateway. For more information, see https://docs.databricks.com/administration-guide/cloud-configurations/aws/customer-managed-vpc.html.
    
    The following parameters only apply if you wish to utilize the [Customer-managed keys for managed services and storage](https://docs.databricks.com/security/keys/customer-managed-keys-managed-services-aws.html) feature.  Note that you must have already created a KMS key to use this template feature:

    * __NewKMSKeyAlias__: AWS KMS key alias for a new KMS key to be used for encrypting managed services, storage, or both.
    * __KeyUseCases__: Configures customer managed encryption keys. Acceptable values are MANAGED_SERVICES, STORAGE, or BOTH. For more information, see https://docs.databricks.com/administration-guide/account-api/new-workspace.html#step-5-configure-customer-managed-keys-optional.
    * __KeyReuseForClusterVolumes__: Only enter a value if the use case is STORAGE or BOTH. Acceptable values are "True" and "False."  
    * __QSS3BucketName__: S3 bucket for Quick Start assets. Use this if you want to customize the Quick Start. The bucket name can include numbers, lowercase letters, uppercase letters, and hyphens, but it cannot start or end with a hyphen (-).
    * __QSS3KeyPrefix__: S3 key prefix to simulate a directory for your deployment assets.  The prefix can include numbers, lowercase letters, uppercase letters, hyphens (-), and forward slashes (/). For more information, see https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingMetadata.html.

    The following parameters are used for tagging purposes:

    * __ResourceOwner__: Used for metadata purposes to keep track of services.  Defaulted to __Databricks_E2__.
    * __ResourcePrefix__: Prefix for tagging purposes.  Defaulted to __DatabricksAccountAPI__.

Each of the resources in the CFT should have comments detailing their purpose.

## Execution

The CFT is larger than the permitted size for using the `--template-body` parameter.  The CFT must be uploaded to a S3 bucket.

To create the stack using the AWS CLI, use the following command, replacing `<aws_profile>` with a profile linked to your AWS account with admin access:

`aws cloudformation create-stack --template-url <path_to_template_file_in_s3> --stack-name DatabricksAWSCFT --profile <aws_profile> --parameters file://params.json --capabilities CAPABILITY_NAMED_IAM`

Several AWS resources will be created.  In addition, several API calls are made through Lambda functions, executing the following steps:

  1. Create a Databricks credentials configuration ID for your AWS role. It calls the Create credential configuration API (POST /accounts/<accountId>/credentials). This request establishes cross-account trust and returns a reference ID to use when you create a new workspace.
  2. Create a storage configuration record that represents the root S3 bucket by calling the [create storage configuration API](https://docs.databricks.com/dev-tools/api/latest/account.html#operation/create-storage-config).
  3. To register your network configuration with Databricks, it calls the [create network configuration API](https://docs.databricks.com/dev-tools/api/latest/account.html#operation/create-network-config).
  4. To register your KMS key with Databricks, it calls the [create customer-managed key configuration API](https://docs.databricks.com/dev-tools/api/latest/account.html#operation/create-notebook-key-config).
  5. To create the new workspace, it calls the [create workspace API](https://docs.databricks.com/dev-tools/api/latest/account.html#operation/create-workspace).

The CFT will provide the following outputs:

  * __CustomerManagedVPCIAMRoleARN__: ARN of the customer managed cross-account IAM role.  If you elected to have this CFT create the cross-account IAM role.
  * __S3BucketName__: Name of the root S3 bucket.
  * __CustomerManagedKeyId__: ID of the customer managed key object.  If you elected to use your KMS key to encrypt data in the control plane and/or data in your AWS account.
  * __CredentialsId__: Credential ID.
  * __ExternalId__: Databricks external ID.
  * __NetworkId__: Databricks network ID.
  * __StorageConfigId__: Storage configuration ID.
  * __WorkspaceURL__: URL of the workspace
  * __WorkspaceStatus__: Status of the requested workspace.
  * __WorkspaceStatusMessage__: Detailed status description of the requested workspace.
  * __PricingTier__: Pricing tier of the workspace. For more information, see https://databricks.com/product/aws-pricing
  * __ClusterPolicyID__: Unique identifier for the cluster policy.

To delete all resources in the stack using the AWS CLI, use the following command, replacing `<stack_name` with the name of your stack and `<aws_profile>` with a profile linked to your AWS account with admin access:

`aws cloudformation delete-stack --stack-name KarveBHTest --profile karve-personal`

## TO DO

This template is a work in progress.  Here is what is left to implement:

  1. Verify PrivateLink configuration is correct.

