# VNET Migration 
## TOC
1. [Summary](#summary)
2. [Prerequisites](#prerequisites)
1. [Credential setup](#credential-setup)
3. [Sample dicovery yaml file](#sample-dicovery-yaml-file)
4. [Pre-discovery](#pre-discovery)
5. [Discovery](#discovery)
6. [Building Aviatrix infrastructure](#building-aviatrix-infrastructure)
7. [Switching the traffic to the Aviatrix transit](#switching-the-traffic-to-the-aviatrix-transit)
8. [Synchronize terraform state with the API based switch changes](#synchronize-terraform-state-with-the-api-based-switch-changes)
9. [Clean up](#clean-up)
10. [Multiple migration cycles](#multiple-migration-cycles)
11. [Back-out changes](#back-out-changes)
12. [Logs](#logs)
13. [S3 bucket](#s3-bucket)
14. [Additional checks](#additional-checks)
15. [Integrating with terraform cloud VCS workflow](#integrating-with-terraform-cloud-vcs-workflow)

## Summary

This script discovers VNET routing info, builds terraform files and allows for switching traffic to the Aviatrix transit

The migration process involves 6 steps:

1. [Preparing the environment](#Prerequisites)

2. [Discovering the VNET(s) configuration](#Discovery)

3. [Building Aviatrix infrastructure](#BuildingAviatrixinfrastructure)

4. [Switching the traffic to the Aviatrix transit](#SwitchingthetraffictotheAviatrixtransit)

5. [Synchronizing terraform state](#SynchronizeterraformstatewiththeAPIbasedswitchchanges)

6. [Deleting unnecessary resources](#Cleanup)

[Logging](#Logs) and [s3 bucket](#S3bucket) details

## Prerequisites
1. Python 3.8+
2. `pip3 install aviatrix-migration`
3. The following permission are required for the script to creating a backup S3 bucket and managing the backup:

      ***"s3:CreateBucket",***\
      ***"s3:DeleteBucket",***\
      ***"s3:ListBucket",***\
      ***"s3:GetObject",***\
      ***"s3:PutObject",***\
      ***"s3:DeleteObject",***\
      ***"s3:GetBucketVersioning",***\
      ***"s3:PutBucketVersioning",***\
      ***"s3:GetBucketPublicAccessBlock"***\
      ***"s3:PutBucketPublicAccessBlock"***

4. Please make sure the following services are registered in your Azure subscription: 
   
   -  At Subscriptions/Resource providers, register **Microsoft.Capacity** for allowing the CPU quota check.
   -  At Subscriptions/Preview features, register **AllowUpdateAddressSpaceInPeeredVnets** for allowing CIDR management on a peered vnet.

## Credential setup

The python migration script requires both the Azure subscription credential and the Aviatrix controller credential for managing cloud resources.  The AWS account credential is OPTIONAL, which is only needed if AWS S3 is configured for backing up the output of the migration script or AWS SSM is used for storing the Aviatrix controller and Azure subscription credentials.

- Azure credential is defined using an Azure application and can be assigned to multiple subscriptions. You input the Azure application and subscription information in the **azure_cred** section of the YAML.  You should define a default application with the name **aviatrix**, to be used by all the subscriptions.  For subscription using a different credential, define additional application with the associated subscription_id explicitly listed.  Here is an example of **azure_cred** section:
  ```
  azure_cred:
    aviatrix:                            ## default Azure credential for all subscriptions
      arm_directory_id: "4780055e-ce37-4f02-b33d-fdad8493a4b6"
      arm_application_id: "b6e09b37-b6cf-49ee-a42f-dcf3946974c1"
      arm_application_secret_env: "ARM_CLIENT_SECRET"
      arm_application_secret_data_src: "avx-azure-client-secret"
    app2:                              #Azure credential for specific subscriptions
      arm_directory_id: "4780055e-ce37-4f02-b33d-fdad8493a4b6"
      arm_application_id: "1234abcd-1234-1234-1234-1234abcd1234"
      arm_application_secret_env: "ARM_CLIENT_SECRET"
      arm_application_secret_data_src: "avx-azure-client-secret"
      arm_subscription_id:
        - "77889900-1234-1234-1234-123456781234"
  ```

  - In this example, subscription 77889900-1234-1234-1234-123456781234 is using app2 credential while all others use the aviatrix credential.
  - The requried parameters defined for an application are arm_directory_id, arm_application_id, arm_application_secret_env, and arm_application_secret_data_src.
  - arm_application_secret_env specifies the name of the environment variable that stores the application secret. This environment variable is read by the migration script.  If you set tf_controller_access to ENV mode, the environment variable TF_VAR_azure_client_secret for storing the same secret is required by terraform; in this case, you can use the same environment variable.
  - arm_application_secret_data_src specifies the name of the AWS SSM data storage for storing the Azure application secret.  It is used by the tf_controller_access SSM mode for customizing the terraform script to read the application secret from the AWS SSM store.
  - The arm_subscription_id attribute expects a subscription_id list.
  - The default application in YAML is named **aviatrix** while other applications can be named with any valid string.

- Controller credential can be defined using the two environment variables:

  ***export aviatrix_controller_user=admin***\
  ***export aviatrix_controller_password=&lt;password&gt;***

  or it can be passed as a command line argument using the --ctrl_user &lt;username&gt; option, where the password would be prompted separately.

- AWS credential can be provided using environment variables AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY or the shared credential file ~/.aws/credentials. In addition, if your migration script is running on a AWS EC2, you can also attach an IAM role with the proper AssumeRole policy.  

Running terraform also requires the Azure application credential and the Aviatrix controller credential to be setup. They can be provided using AWS SSM storage or environment variables, depending on the mode settings in the tf_controller_access section of the YAML file:
  - In SSM mode, both controller password and azure credential are stored in AWS SSM. You specify in the tf_controller_access section the name and location of the SSM store (in the AWS controller account), and the AWS provider alias and ssm_role for accessing the controller secret.  The SSM name of the Azure application secret is defined by the attribute arm_application_secret_data_src in azure_cred section.  In this case, the AWS account credential is also needed by terraform and is read using the same approach (environment variables, shared credential file, or instance profile) as described for the migration script.

  - In ENV mode, controller credential is passed to the terraform using the two environment variables AVIATRIX_USERNAME and AVIATRIX_PASSWORD while Azure application secret is passed using the environment variable TF_VAR_azure_client_secret. In this case, the AWS account credential is NOT needed.


## Sample dicovery yaml file
```
       label: "AZURE"
 #     aws:
 #       s3: 
 #         name: "discovery-migration-terraform1"
 #         region: "us-east-2"
 #         account: "205987878622"
 #         role_name: "aviatrix-role-app"
       terraform:
         # regenerate_common_tf: True
         terraform_output: "/Users/ybwong/MY_STUFF/tfoutput"
         terraform_version: ">= 0.14"
         aviatrix_provider: "= 2.19.4"
         arm_provider: "~> 2.46.0"
         aws_provider: "~> 3.43.0"
         # account_folder: "id"                             ## available options are "name" or "id". Default is "id"
         # enable_s3_backend: False
         # module_source: "../abc"               ## use a module with a different name or location
         # module_name: "abc"                    ## name of the module generated under the terraform_output folder
         # tf_cloud:                             ## terraform cloud integration
         #   organization: "aviatrix-ps-test"    ## organization is mandatory
         #   workspace_name: "terraform-cli-ws"  ## either workspace_name or tags is allowed
         #   tags: ["abc"]                       ## This information will be written to the cloud block in terraform
         # tmp_folder_id: "YAML_ID"              ## If omitted, tmp folder is not tagged. This is the current behavior.
                                                 ## YAML_ID: tag tmp folder with yaml filename
                                                 ## VNET_ID: tag tmp folder with the first VNET ID found.
       azure_cred:
         aviatrix:                                          ## default Azure credential for all subscriptions
           arm_directory_id: "4780055e-ce37-4f02-b33d-fdad8493a4b6"
           arm_application_id: "b6e09b37-b6cf-49ee-a42f-dcf3946974c1"
           arm_application_secret_env: "ARM_CLIENT_SECRET"
           arm_application_secret_data_src: "avx-azure-client-secret"
         # app2:                                            ## Azure credential for specific subscriptions
         #  arm_directory_id: "4780055e-ce37-4f02-b33d-fdad8493a4b6"
         #  arm_application_id: "1234abcd-1234-1234-1234-1234abcd1234"
         #  arm_application_secret_env: "ARM_CLIENT_SECRET"
         #  arm_application_secret_data_src: "avx-azure-client-secret"
         #  arm_subscription_id:
         #    - "77889900-1234-1234-1234-123456781234"
       aviatrix:
         controller_ip: "52.3.72.231"
         # tf_controller_access:                ## Default to "ENV". Available options are "SSM" or "ENV"
         #   mode: "SSM"                        ## When using ENV, the rest of the attributes are not needed.
         #   alias: "us_west_2"
         #   region: "us-west-2"
         #   username: "admin"  
         #   password_store: "avx-admin-password"
         #   ssm_role: ""                       ## Default to "".  If defined with a role, generate assume_role statmeent in provider for SSM access; otherwise, no assume_role statement is generated.
         #   account_id: ""                     ## AWS controller account id.  Default to "", only required if ssm_role is defined.
 #     alert:
 #        vnet_name_length: 31
 #        vnet_peering: True
 #        vcpu_limit: True
 #        check_gateway_existence: True         ## ## Check if a spoke gateway has been deployed
       config:
         configure_gw_name: True
         # route_table_tags:
         #  - key: "Aviatrix-Created-Resource"
         #    value: "Do-Not-Delete-Aviatrix-Created-Resource"
         # configure_private_subnet: False
         # configure_staging_attachment: False  ## If set to True, attach the spoke gateway to transit at staging time. Default is True
       account_info:
         - subscription_id: "6f1b55d5-4e5a-4166-8175-cfc4d58d2515"
           # tf_provider_alias: "s0_sub_core_01"
           account_name: "arm_dev"            
           # onboard_account: False
           # spoke_gw_size: "Standard_D3_v2"
           # hpe: True
           # max_hpe_performance: True         ## use max tunnel only valid if hpe is True
           # vwan:
           #   subscription_id: "77889900-1234-1234-1234-123456781234"
           #   resource_group: "rg-east-us"
           #   vhub: "vhub-east-us"
           vnets:
             - vnet_name: "vn_firenet-test_VPC1-US-East"
               avtx_cidr: "10.112.0.0/16"            
               use_azs: True
               spoke_gw_name: "azu-spoke-1"
               transit_gw_name: "azu-use-transit-gw"
               # spoke_gw_tags:
               #  - key: ""
               #    value: ""
               # inspection: True       ## Default is False if omitted.
               # domain: "prod"         ## Default is None
               # copy_quad_zero_route   ## Default is False
             - vnet_name: "vn_firenet-test_VPC2-US-East"
               avtx_cidr: "13.1.1.0/24"
               use_azs: True
               spoke_gw_name: "azu-spoke-2"
               transit_gw_name: "azu-use-transit-gw"
 #     prestage:
 #       default_route_table: "dummy_rt"              ## Temporary route table name used in pre-stage
 #     switch_traffic:
 #       transit_peerings:
 #         azu-usw-transit-gw: "aws-usw1-transit-gw"  ## Used to detect readiness of the transit gateway with on-prem connection when migrating from transit disconnected from on-prem
 #       delete_vnet_lock: True
 #       delete_peering: True                        ## Delete vnet peerings at switch_traffic, False by default.
 #       manage_terraform_import: True               ## switch_traffic will manage the import/undo-import of terraform_import_associations.sh.  Default is False.
 #     cleanup:
 #       resources: []                                ## available options are ["PEERING", "VNG_ER"]
 #       delete_vnet_lock: True
```

| Field              |  Type         | Required | Description                                      |
| :-----------       |:------------- | :------- | :--------------------------------------------    |
| label:             | string        |   Yes    | Mark this yaml for Azure discovery               |
|                    |               |          |                                                  |
| aws:               |               |   No     | Use AWS S3 to backup the generated account folder. Omit this section if S3 backup is not required. |
| s3:                |               |          | Setup s3 for storing the terraform output files  |
| account:           | string        |          | s3 bucket account number                         |
| role_name:         | string        |          | s3 bucket access permission                      |
| name:              | string        |          | s3 bucket name                                   |
| region:            | string        |          | s3 bucket region                                 |
|                    |               |          |                                                  |
| terraform:         |               |   Yes    | Mark the beginning of terraform info             |
|regenerate_common_tf| bool          |   No     | True by default. Decide whether common terraform files in account folder should be regenerated every time when discovery is run.|
| terraform_output   | string        |   Yes    | Absolute path to the TF files created            |
| account_folder     | string        |   No     | Specify whether account_folder should be named using account name or subscription id. Valid options are "name" or "id" Default is "id" if omitted.            |
| terraform_version  | string        |   Yes    | Version string in terraform version syntax       |
| aviatrix_provider  | string        |   Yes    | Version string in terraform version syntax       |
| arm_provider       | string        |   Yes    | Version string in terraform version syntax       |
| aws_provider       | string        |   Yes    | Version string in terraform version syntax       |
| enable_s3_backend  | bool          |   No     | Generate backend s3 terraform resource or not. Default is False. |
| module_source      | string        |   No     | Override the module source in vnet.tf. Default is "../module_azure_brownfield_spoke_vnet" if omitted  |
| module_name        | string        |   No     | Define the name of the module generated under the terraform_output folder. Default is "module_azure_brownfield_spoke_vnet". |
| tf_cloud:          |               |   No     | terraform cloud integration                      |
| organization       | string        |   Yes    | This is a mandatory attribute for terraform cloud integration |
| workspace_name     | string        |   Yes    | Either workspace_name or tags should be defined, not both.  |
| tags               | List of string|   Yes    | This information will be used to define the cloud block in terraform |
| tmp_folder_id      | string        |   No     | If omitted, tmp folder is not tagged. "YAML_ID": tag tmp folder with yaml filename. "VNET_ID": tag tmp folder with the first VNET ID found. |
|                    |               |          |                                                  |
| azure_cred:        |               |   Yes    | Mark the beginning of Azure Application credential|
| aviatrix:          |               |   Yes    | Define the default credential ("aviatrix" application) for all subscriptions. |
| arm_directory_id   |               |   Yes    | Also referred to as the tenant id                |
| arm_application_id |               |   Yes    |                                                  |
|arm_application_secret_env|         |   Yes    | Environment variable name that holds the application secret. Used by migration script. |
|arm_application_secret_data_src|    |   No     | Default to "avx-azure-client-secret". Name of the AWS SSM storage that holds the application secret|
|                    |               |          |                                                  |
| app2:              |               |   No     | Define additional credentials for other subscriptions |
| arm_directory_id   |               |   No     | Also referred to as the tenant id                |
| arm_application_id |               |   No     |                                                  |
|arm_application_secret_env|         |   No     | Environment variable name that holds the application secret. Used by migration script.|
|arm_application_secret_data_src|    |   No     | Default to "avx-azure-client-secret". Name of the AWS SSM storage that holds the application secret|
|arm_subscription_id | List          |   No     | List of subscription IDs that is associated to this application|
|                    |               |                                                             |
| aviatrix:          |               |   Yes    | Mark the beginning of aviatrix info              |
| controller_ip      | string        |   Yes    | Aviatrix controller IP address                   |
|                    |               |          |                                                  |
|tf_controller_access|               |   No     | Mark the beginning of tf_controller_access inside terraform section. See [tf_controller_access](#tf-controller-access) for details. |
| mode               | string        |   No     | Default to "ENV". Available options are "SSM" and "ENV" |
| alias              | string        |   No     | Default to "us_west_2". Used only in SSM mode.   |
| region             | string        |   No     | Default to "us-west-2". Used only in SSM mode.   |
| password_store     | string        |   No     | Default to "avx-admin-password". Used only in SSM mode. |
| username           | string        |   No     | Default to "admin".  Used only in SSM mode.      |
| ssm_role           | string        |   No     | Default to "". Used only in SSM mode. |
| account_id         | string        |   No     | AWS Account id where the controller is deployed. Default to "". Required if ssm_role is defined. |
|                    |               |          |                                                  |
| alert:             |               |   No     | Mark beginning of alert                          |
| ​vnet_name_length   | int           |   No     | Alert VNET name length longer than given value. Set to 31 by default. Set to 0 to disable alert.|
| ​vnet_peering       | bool          |   No     | Alert VNET peering. True by default.  Set to False to disable alert. |
| ​vcpu_limit         | bool          |   No     | Check vCPU quota limit. True by default. Set to False to disable alert. Requires Microsoft.Capacity registration. |
| check_gateway_existence | bool     |   No     | True by defaut if omitted. Check if a spoke gateway has been deployed in the VPC. |
|                    |               |          |                                                  |
| config:            |               |   No     | Mark beginning of script feature config          |
| configure_gw_name  | bool          |   No     | True by default.                                 |
| route_table_tags   | list          |   No     | List of tags to be added to the route table(s). Omit this section if no tags required.   |
| - key              | String        |   No     |  name of the tag                                 |
| - value            | String        |   No     |  value of the tag                                |
| configure_private_subnet | bool    |   No     |  Turn all subnets into private subnet for egress purpose. Default is False. (See [configure_private_subnet](#configure-private-subnet) for details.)|  
| configure_staging_attachment | bool|   No     | Default is True. Configure the module_azure_brownfield/main.tf to attach spoke to transit at  staging. |
|                    |               |          |                                                  |
| account_info:      |               |   Yes    | Spoke VNET info                                  |
| subscription_id    | string        |   Yes    | Azure subscription #                             |
| account_name       | string        |   Yes    | Aviatrix account name                            |
| tf_provider_alias  | string        |   No     | Default: Use value of account_name as alias      |
| onboard_account    | bool          |   No     | Default is False. Determine whether to generate the terraform resource for onboarding Aviatrix account.|
| spoke_gw_size      | string        |   No     | Spoke gateway instance size. Default is set to "Standard_D3_v2". |
| hpe                | bool          |   No     | Hpe mode. True by default.  The same attribute is available at the vnet level   |
| max_hpe_performance| bool          |   No     | Number of tunnels used in spoke attachment. True by default, build attachment with maximum number of tunnels allowed. Use only one tunnel if set to False. This attribute is ONLY valid on and after aviatrix_provider 2.22.3 and when hpe is True.  The same attribute is available at the vnet level.  |
| vwan:              |               |   No     | Virtual wan section. If omitted, the spoke vnet subscription will be used to search for the vwan  |
| subscription_id    | string        |   No     | The subscription where vwan belongs              |
| resource_group     | string        |   No     | The resource group where vwan belongs            |
| vhub               | string        |   No     | The name of the vhub                             |
|                    |               |          |                                                  |
| vnets:             |               |   Yes    | Mark the beginning of vnet list                  |
| vnet_name          | string        |   Yes    | VNET to be migrated                              |
| use_azs            | bool          |   No     | True by default.                                 |
| avtx_cidr          | string        |   Yes    | set avtx_cidr in vnet.tf                         |
| domain             | string        |   No     | Default is None.                                 |
| hpe                | bool          |   No     | Vnet-level Hpe mode. Not defined by default. If defined, it will override the same attribute defined  at the account level.  |
| inspection         |  bool         |   No     | Default is False.  No inpsection required        |
| max_hpe_performance| bool          |   No     | Vnet-level max_hpe_performance. Number of tunnels used in spoke attachment. Not defined by default. If defined, it will override the same attribute defined at the account level.         |
| copy_quad_zero_route | bool        |   No     | Default is False. Will not copy quad-zero virtual appliance route. |
| spoke_gw_name      | string        |  Yes     | Not required if configure_gw_name is set to False     |
| transit_gw_name    | string        |  Yes     | Not required if configure_gw_name is set to False     |
| spoke_gw_tags      |               |    No    | mark beginning of a list, following by a list of key and value object|
|  - key             | String        |   No     |  name of the tag                                 |
|  - value           | String        |   No     |  value of the tag                                |
|                    |               |          |                                                  |
| prestage:          |               |   No     |  Only if prestaging is required.                 |
| default_route_table| string        |          |  Temporary route table name used in pre-stage. Set to "dummy_rt" by default.  Final RT name <vnet_name>-<default_route_table>.  |
|                    |               |          |                                                  |
| switch_traffic     |               |   No     | Mark the beginning of switch_traffic             |
| transit_peerings   |               |   No     | Azure transit to AWS s2c transit peering map.  Used to detect readiness of the transit gateway with on-prem connection when migrating from transit disconnected from on-prem .    |
| delete_vnet_lock   | bool          |   No     | Indicate whether vnet lock in spoke vnet and transit vnet should be deleted before the operation. False by default. |
| delete_peering     | bool          |   No     | Delete vnet peerings at switch_traffic, False by default. |
| manage_terraform_import | bool     |   No     | switch_traffic will manage the terraform import/undo-import of terraform_import_associations.sh. Default is False. |
|                    |               |          |                                                  |
| cleanup:           |               |   Yes    | Mark the beginning of cleanup list               |
| resources          | list ["PEERING", "VNG_ER"]  |   Yes    | Delete resource in VNET.  Use an empty list [] to omit deletion.    |
| delete_vnet_lock   | bool          |   No     | Indicate whether vnet lock in spoke vnet should be deleted before the operation. False by default. |

  **label**: Since there are different type of YAML inputs, such as YAML for AWS migration, vnet CIDR management, and Azure migration, the label "AZURE" attribute is used to identify the YAML type so the migration script can validate accordingly.

  **aws/s3**: This section defines the AWS S3 attributes for backing up the terraform output folder so the terraform files can be restored when needed.

  **azure_cred** defines the Azure application credential.  The default application "aviatrix" credential will be used for all subscriptions by default.  Additional application credential with any name can be defined and assigned to the specific subscriptions by using the arm_subscription_id attribute as shown in "app2".  Subscription with ID explicitly listed under arm_subscription_id will use the associated application credential, instead of the default aviatrix.

  **account_info** list the spoke vnets to be migrated in each subscription denoted by subscription_id.  The **account_name** attribute serves multiple purposes. 1) It defines the folder name if **terraform/account_folder** is set to "name". 2) It is the account name used in the controller if **onboard_account** is set to True.  3) It is also the default provider alias if **tf_provider_alias** is not defined.

  <a id="tf-controller-access"></a>
  **tf_controller_access**: This is a subsection inside terraform config. Two modes are supported: 
  
  - SSM. Terraform will retrieve controller password and Azure client secret from AWS SSM. The attributes under tf_controller_access are used to configure the provider and data access for SSM. To grant SSM permission, one can set ssm_role to a role with the ssm:GetParameter permission, e.g., aviatrix-role-app.  If defined, an assume_role statement is generated in the terraform AWS provider; otherwise, no assume_role statement is generated.
  - ENV. Use environment variables AVIATRIX_USERNAME and AVIATRIX_PASSWORD to pass controller credential into terraform. Use environment variable TF_VAR_azure_client_secret to pass the Azure client secret into terraform.

  <a id="configure-private-subnet"></a>
  **configure_private_subnet**: Turn all subnets into private subnet for Aviatrix Egress. It adds a quad-zero-to-none route into the copied route tables, including main-1 and main-2 created for subnets without a route table.  These quad-zero routes will be replaced by the controller to point to the spoke gateway ENI at dm.arm.switch_traffic.

  <a id="configure-staging-attachment"></a>
  **configure_staging_attachment**: Set to True to add spoke attachment at staging.  Set to False to skip spoke attachment.  When operated in HPE mode with Firewall inside Azure transit vnet, this is a workaround to prevent the peering CIDR of Aviatrix's native peering from spreading out of the Aviatrix transit to the Azure transit vnet, where the Firewall is located.
  If not, outgoing traffic of a non-migrated vnet will go through the Firewall and then to the Aviatrix transit to reach a staged vnet (with attached spoke gateway before switch traffic) while the return traffic will go directly to the Azure transit vnet and dropped by the Firewall.
  
## Pre-discovery

Two additional steps are required for Azure migration before running discovery:

1. Add **avtx_cidr** to VNET using **dm.arm.mgmt_vnet_cdir**:\
   ***python -m dm.arm.mgmt_vnet_cidr discovery.yaml***\
   <br>
   This is a helper script for VNET CIDR management. It uses the Azure management.azure.com REST API 2021-02-01 to update the CIDR and sync the remote peered vnet. At the time of writing this documentation, adding CIDR to a VNET with peering is still an Azure preview feature.  Go to your **subscription/Preview features**, register the preview feature **AllowUpdateAddressSpaceInPeeredVnet** in your subscription to activate it.   Here are the possible usages of the command:
   
   - To add cidrs specified in yaml:\
     ***python -m dm.arm.mgmt_vnet_cidr cidr.yaml***
   - To delete cidrs:\
     ***python -m dm.arm.mgmt_vnet_cidr cidr.yaml --delete***

   This tool has been enhanced to support vWAN, i.e., vnet-to-vHub peering.
   
2. Run **dm.arm.prestage** to assign an empty route table to the subnets without route table association, so the controller will not create one when attaching the spoke gateway. This would allow the migration process to have full control of the UDR route table and manage its properties.  This is required until the controller can expose the management API for the UDR route tables it creates.  Here are the various usages of **dm.arm.prestage**:

   - Create the empty route table and asssociate subnet (without route table) to it.\
     ***python3 -m dm.arm.prestage --yaml_file discovery.yaml***
   - Revert the pre-staging process, deleting the subnet associations and the route table.\
     ***python3 -m dm.arm.prestage --revert --yaml_file discovery.yaml***
   - Perform a dry run and see what would be done.\
     ***python3 -m dm.arm.prestage --dry_run --yaml_file discovery.yaml***

   <br>
   Define the route table name using the **default_route_table** attribute in the discovery yaml:

         prestage:
            default_route_table: "dummy_rt"
   
   - The final route table will have a name of the following form:\
     &lt;vnet_name&gt;-&lt;default_route_table&gt;

## Discovery
1. Provide [discovery info](#Sample-dicovery-yaml-file) in the discovery.yaml file

2. Run the discovery script to review and resolve the reported alerts \
   ***python3 -m dm.arm.discovery azure_discovery.yaml***

   - The script generates terraform files required to build Aviatrix infrastructure in the migrated VNET. They can be found in **&lt;terraform_output&gt;/&lt;subscription Id&gt;**.

   - The script shows progress by printing to stdout the discovered VNET and detected alerts.  It produces two log files: **dm.log** contains the results of the checks and tasks that are done for each VNET; **dm.alert.log** captures the alert summary that are seen on stdout.  They can be found in **&lt;terraform_output&gt;/log**.

3. After resolving the alerts, run discovery again with the option --s3backup to backup the **&lt;terraform_output&gt;/&lt;subscription Id&gt;** folder into the S3 bucket **&lt;bucket&gt;/dm/&lt;subscription Id&gt;**: \
   ***python3 -m dm.arm.discovery discovery.yaml --s3backup***

   This will allow the terraform files to be re-called later in switch_traffic time where new subnet-route-table association will be generated and appended to the existing terraform files.

**Migration restriction**\
  A single discovery.yaml file can be used to migrate all VNETs within an account in a single migration (discovery/staging/switch_traffic/cleanup) cycle, or multiple yaml files can be used to migrating a group of VNETs at a time, i.e., in seperate migration cycles.  There is one restriction: On the SAME account, a migration cycle must be completed before the next one can be started, i.e., migration cycle cannot be overlapped within the same account.

## Building Aviatrix infrastructure

1. Copy terraform directory from the ***terraform_output*** folder to the ***azure_spokes*** one\
    The terraform files contain the Aviatrix infrastructure  resources, the discovered
    subnets and copied route tables.  The subnet resources should be imported before building the Aviatrix infrastructure .  
    
    The following steps should be performed in the ***subscription Id*** directory:

2. Run ***terraform init***

3. Optional - Run terraform apply on the following two targets to create TF state file if none exists. These steps are only applicable if tf_controller_acess is set to SSM mode:\
  ***terraform apply -target data.aws_ssm_parameter.avx-azure-client-secret***\
  ***terraform apply -target data.aws_ssm_parameter.avx-password***

4. Run ***source ./terraform-import-subnets.sh*** to import discovered subnet resources.\
   To undo the import, run ***source ./tmp/terraform-undo-import-subnets.sh***.

5. Run ***terraform apply*** to build Aviatrix infrastructure . Terraform will deploy and attach the spoke gateways.\
   It will also copy existing route tables.

## Switching the traffic to the Aviatrix transit
1. Run the switch_traffic command to switch traffic from Azure VNET to the Aviatrix transit. \
    This command has two forms, depending whether you want to download the discovery.yaml from S3
(archived during the discovery phase) or use a local input yaml file:
      
      - Download discovery.yaml from the S3 bucket to /tmp/discovery.yaml.  The is the typical flow.\
        ***python3 -m dm.arm.switch_traffic --ctrl_user &lt;controller-admin-id&gt; --s3_yaml_download &lt;s3_account&gt;,&lt;s3_account_role&gt;,&lt;s3_bucket&gt;,&lt;spoke_account&gt;***        

      - Use a local yaml file. \
        ***python3 -m dm.arm.switch_traffic --ctrl_user &lt;controller-admin-id&gt; --yaml_file discovery.yaml***

    **dm.arm.switch_traffic** is responsible for:\
      a) changing the subnets association to the new RTs created in the Building Aviatrix infrastructure phase\
      b) setting up the VNET advertised CIDRs in the Aviatrix spoke gateways.
      
    It supports the following additional options:

      - ***--s3backup*** uploads the associated terraform output account folder into S3 at the end of switch_traffic, e.g.:\
      ***python3 -m dm.arm.switch_traffic --s3backup --ctrl_user &lt;controller-admin-id&gt;  --s3_yaml_download &lt;s3_account&gt;,&lt;s3_account_role&gt;,&lt;s3_bucket&gt;,&lt;spoke_account&gt;***

      - ***--s3download*** downloads the account folder, containing the terraform script, from S3 at the beginning of switch_traffic. This is required if the local account folder has been removed because of a container restart, e.g.:\
      ***python3 -m dm.arm.switch_traffic --s3download --ctrl_user &lt;controller-admin-id&gt;  --s3_yaml_download &lt;s3_account&gt;,&lt;s3_account_role&gt;,&lt;s3_bucket&gt;,&lt;spoke_account&gt;***

      - ***--revert*** restores the configuration back to its original state.\
      It includes restoring the original subnet and route table associations and removing all the spoke gateway advertised CIDRs, e.g.:\
      ***python3 -m dm.arm.switch_traffic -ctrl_user &lt;controller-admin-id&gt; --revert --s3_yaml_download &lt;s3_account&gt;,&lt;s3_account_role&gt;,&lt;s3_bucket&gt;,&lt ;spoke_account&gt;***

      - ***--dry_run*** runs through the switch_traffic logic and reports the detected alerts and list of changes to be made **without** actually making any changes, e.g.:\
      ***python3 -m dm.arm.switch_traffic --dry_run --ctrl_user &lt;controller-admin-id&gt; --s3_yaml_download &lt;s3_account&gt;,&lt;s3_account_role&gt;,&lt;s3_bucket&gt;,&lt;spoke_account&gt;***

    If the environment variables for controller username and password are defined (see [Prerequisites](#prerequisites), Step 5), you can run the **switch_traffic**  without the **--ctrl_user** option; in which case, the migration script will not prompt you for the controller password and use the environment variable defined credential instead, e.g.:\
    ***python3 -m dm.arm.switch_traffic --s3_yaml_download &lt;s3_account&gt;,&lt;s3_account_role&gt;,&lt;s3_bucket&gt;,&lt;spoke_account&gt;*** 

## Synchronize terraform state with the API based switch changes
1. Copy terraform directory from the ***terraform_output/&lt;subscription Id&gt;*** folder to the ***aws_spokes*** folder again because **switch_traffic** will append new route-table-to-subnet-association resources to the corresponding ***&lt;vnet-name&gt;.tf*** in the ***terraform_output/&lt;subscription Id&gt;*** folder.

2. Import new subnet-association resources into terraform\
   Run ***source ./terraform-import-associations.sh*** at the ***azure_spoke*** folder.\
   To undo the import, run ***source ./tmp/terraform-undo-import-associations.sh***.



## Clean up
1. Run the following command to delete original UDR route tables and peerings. This command has two forms, depending whether you want to download the discovery.yaml from S3
(archived at discovery time) or use a local input yaml file: 

      - Download discovery.yaml from S3 to /tmp/discovery.yaml.  The is the typical flow.\
        ***python3 -m dm.arm.cleanup --s3_yaml_download &lt;s3_account&gt;,&lt;s3_account_role&gt;,&lt;s3_bucket&gt;,&lt;spoke_account&gt;***        

      - Use a local yaml file. \
        ***python3 -m dm.arm.cleanup --yaml_file discovery.yaml***

    **dm.arm.cleanup** is responsible for\
      a) deleting the revert.json from tmp folder locally and from s3.\
      b) deleting the original route tables\
      c) deleting the Peering
      
    Cleanup process can be re-run multiple times. 
    It also has a dry_run option: 

      ***--dry_run*** allows the users to review the resources to be deleted before the actual clean up, e.g.:

      - Download discovery.yaml from S3 to /tmp/discovery.yaml.  The is the typical flow.\
      ***python3 -m dm.arm.cleanup --dry_run --s3_yaml_download &lt;s3_account&gt;,&lt;s3_account_role&gt;,&lt;s3_bucket&gt;,&lt;spoke_account&gt;***

      - Use a local yaml file. \
      ***python3 -m dm.arm.cleanup --dry_run --yaml_file discovery.yaml***


## Multiple migration cycles
Each migration is associated with a YAML file and is made up of a couple of steps: discovery, staging, switch traffic and cleanup. There are some temporary files generated in the process and kept in tmp folder inside the account folder, e.g.:
```
  log
  module_azure_brownfield_spoke_vnet
  - 72d1c01e-09cc-4ff1-b0ea-11d708943bbc
    simplest-vnet-3.auto.tfvars
    simplest-vnet-3.tf
    main.tf
    providers.tf		
    variables.tf
    versions.tf
    terraform.tfvars
    terraform-import-subnets.sh
    terraform-import-associations.sh
    - tmp
      discovery.yaml
      revert.json
      terraform-undo-import-subnets.sh
      terraform-undo-import-associations.sh
```

Problem occurs when multiple migration are running at the same time and are writing to the same output folder, because the files kept in the tmp folder will be overwritten by different migration.  There are two ways to support multiple migration cycles, preventing temporary files from being overwritten.

1. Use a separate output location (specified with the terraform_output attribute) for each migration.  This naturally isolates the local terraform state and the related temporary files of a migration.  In this case, each migration is self-contained and can be managed separately.

1. Tag the tmp folder uniquely. If the same terraform_output folder is used for multiple migrations, we can tag the tmp folder uniquely for each migration. So each migration will operate in its own tmp folder.  The tmp_folder_id attribute in the YAML terraform section is introduced for this purpose:

   1. If tmp_folder_id is omitted, we fall back to the existing behavior, i.e., no tagging. If tmp_folder_id is set to VNET_ID, we pick the first VNET ID in the first subscription in the YAML as the tag, regardless of the number of subscriptions in the YAML, e.g., tmp-simplest-vnet-3. If tmp_folder_id is set to YAML_ID, we tag the tmp folder with the YAML file name. For example, if input YAML file is abc.yaml, the tmp folder name would be tmp-abc-yaml.

   1. To avoid creating too many folders directly under the account folder, all these tagged tmp folders will be created as a sub-folder under the current tmp folder. Here is an example of  the folder structure when setting tmp_folder_id to YAML_ID and with xyz.yaml as input.
   ```
      log
      module_azure_brownfield_spoke_vnet
      - 72d1c01e-09cc-4ff1-b0ea-11d708943bbc
        simplest-vnet-3.auto.tfvars
        simplest-vnet-3.tf
        main.tf
        providers.tf		
        variables.tf
        versions.tf
        terraform.tfvars
        terraform-import-subnets.sh
        terraform-import-associations.sh
        - tmp
          - tmp-xyx-yaml
            terraform-import-subnets.sh
            terraform-undo-import-subnets.sh
            discovery.yaml
            revert.json
            terraform-undo-import-associations.sh
            terraform-import-associations.sh            
    ```

1. In the new tagged tmp folder, we also keep a copy of the associated terraform-import-subnets.sh and terraform-import-associations.sh scripts.


## Back-out changes
After tagging the tmp folder, dm.arm.switch_traffic would be able to revert the traffic of each individual migration. The associated terraform-undo scripts kept in each tagged tmp folder would be able to remove the imported resources of the associated migration from the terraform state. The only question is how to destroy the physical resources of a specific migration, if needed. Note, terraform destroy will indiscriminately remove all the resources it kept in the terraform state, but we only want to remove a specific migration dictated by a YAML file.

The solution to this problem is to tag (rename) the related vnet-id.tf and vnet-id.auto.tfvars files to vnet-id.tf.destroy and vnet-id.auto.tfvars.destroy.  After that, run "terraform APPLY". The end result in terraform is a deletion of the resources in the missing vnet-id.tf files.  The new helper tool **tfhelper** is introduced for this tagging purpose:

```
python -m dm.arm.tfhelper -h
usage: tfhelper.py [-h] [--dry_run] [--restore] [--pre_destroy] [--post_destroy] [--vnet VNET] [--s3download] [-f] yaml_file_path

A helper tool for managing account folder

positional arguments:
  yaml_file_path

optional arguments:
  -h, --help      show this help message and exit
  --dry_run       Dry run tfhelper logic and show what will be done
  --restore       Unmarked the destroyed <vnet-id>.tf files
  --pre_destroy   Marked the <vnet-id>.tf files to be destroyed
  --post_destroy  Remove the backed out <vnet-id>.tf and <vnet-id>.auto.tfvars files
  --vnet VNET     Back-out a specific vnet
  --s3download    Download the archived account folder from S3
  -f, --force     force pre_destory
```

  - To remove the resources (or tag all the VNETs) in the current YAML, e.g., a.yaml:\
    ***python3 -m dm.arm.tfhelper --pre_destroy a.yaml
  - After that, run terraform apply

To illustrate the process, let's assume that we have two separate migrations going on with the same terraform_output location.  The first migration is based upon a.yaml and the second one b.yaml. Let's further assume that the following steps have been done in sequence, steps 1 to 11 for the first migration and 12 to 20 the second migration:

1. python -m dm.arm.mgmt_vnet_cidr a.yaml
1. python -m dm.arm.prestage --yaml_file a.yaml
1. python -m dm.arm.discovery a.yaml
1. terraform init
1. terraform apply -target data.aws_ssm_parameter.avx-password
1. terraform apply -target data.aws_ssm_parameter.avx-azure-client-secret
1. source terraform-import-subnets.sh
1. terraform apply
1. python -m dm.arm.switch_traffic --yaml_file a.yaml
1. source terraform-import-associations.sh
1. terraform apply
1. python -m dm.arm.mgmt_vnet_cidr b.yaml
1. python -m dm.arm.prestage --yaml_file b.yaml
1. python -m dm.arm.discovery b.yaml
1. terraform init
1. source terraform-import-subnets.sh
1. terraform apply
1. python -m dm.arm.switch_traffic --yaml_file b.yaml
1. source terraform-import-associations.sh
1. terraform apply

To backout the changes of the first migration only, here are the steps to follow:
1. source tmp/tmp-a-yaml/terraform-undo-import-associations.sh
1. python -m dm.arm.switch_traffic --yaml_file a.yaml --revert
1. source tmp/tmp-a-yaml/terraform-undo-import-subnets.sh
1. python -m dm.arm.tfhelper --pre_destroy a.yaml
1. terraform apply
1. python -m dm.arm.prestage --yaml_file a.yaml --revert
1. python -m dm.arm.mgmt_vnet_cidr a.yaml --delete

There is a --post_destroy option you can use as step 8 to delete the related vnet-id.destroy and vnet-id.auto.tfvars.destroy for the given YAML, e.g., python -m tfhelper --post_destroy a.yaml. You can use the --restore option to rename the related vnet-id.tf.destroy and vnet-id.auto.tfvars back to vnet-id.tf and vnet-id.auto.tfvars, i.e., removing the .destroy tag.


## Logs

Both **dm.arm.discovery** and **dm.arm.switch_traffic** append its log to the end of the two log files: **dm.log** logs the details resulting from the checks and tasks that are done per VNET. **dm.alert.log** captures the same alert summary that are seen on stdout. Here is an example log that shows the beginning of each **dm.arm.discovery** run:

      2021-08-28 00:15:44,618 #############################################
      2021-08-28 00:15:44,618 
      2021-08-28 00:15:44,618 dm.arm.discovery azure.yaml
      2021-08-28 00:15:44,618 
      2021-08-28 00:15:44,618 #############################################
      2021-08-28 00:15:44,618 
      2021-08-28 00:15:44,618   **Alert** Failed to read S3 name attribute in yaml
      2021-08-28 00:15:44,618   **Alert** Failed to read S3 attribute(s) in yaml
      2021-08-28 00:15:44,624 +++++++++++++++++++++++++++++++++++++++++++++
      2021-08-28 00:15:44,624 
      2021-08-28 00:15:44,624     Subscription ID :  6f1b55d5-4e5a-4166-8175-cfc4d58d2515
      2021-08-28 00:15:44,625 
      2021-08-28 00:15:44,625 +++++++++++++++++++++++++++++++++++++++++++++
      2021-08-28 00:15:44,625 
      2021-08-28 00:15:45,023 - Discover route table without subnet association
      2021-08-28 00:15:45,710   **Alert** westus rt_firenet-test_VPC1-EU-West.subnet1 has no subnet association
      2021-08-28 00:15:45,710   **Alert** westus rt_firenet-test_VPC1-EU-West.subnet2 has no subnet association
      2021-08-28 00:15:45,710   **Alert** westus rt_firenet-test_VPC1-EU-West.subnet3 has no subnet association
      2021-08-28 00:15:45,710   **Alert** westus test-route-table has no subnet association
      2021-08-28 00:15:45,710   **Alert** northeurope rt-test-north-europe has no subnet association
      2021-08-28 00:15:45,710   **Alert** southeastasia rt-test-southeast-asia has no subnet association
      2021-08-28 00:15:45,711   **Alert** centralus rtb-private has no subnet association
      2021-08-28 00:15:45,711   **Alert** japaneast rt-test-japan-test has no subnet association
      2021-08-28 00:15:45,711   **Alert** westus2 rt-test-west-us-2 has no subnet association
      2021-08-28 00:15:46,221 - Discover vnets: 7 vnets
      2021-08-28 00:15:46,221   ...............................................................
      2021-08-28 00:15:46,221   Vnet cidr                Vnet name (rg)           
      2021-08-28 00:15:46,221   ...............................................................
      2021-08-28 00:15:46,221   10.1.0.0/16              vn_firenet-test_VPC1-US-East  (rg_firenet-test_eastus)
      2021-08-28 00:15:46,222   10.137.24.0/21           vn_firenet-test_VPC2-US-East  (rg_firenet-test_eastus)
      2021-08-28 00:15:46,718 - check if vn_firenet-test_VPC1-US-East has been migrated
      2021-08-28 00:15:47,445   no spoke gateway found
      2021-08-28 00:15:47,445 
      2021-08-28 00:15:47,446 ---------------------------------------------
      2021-08-28 00:15:47,446 
      2021-08-28 00:15:47,446     Vnet Name : vn_firenet-test_VPC1-US-East
      2021-08-28 00:15:47,446     CIDRs     : ['10.1.0.0/16', '10.112.0.0/16']
      2021-08-28 00:15:47,446     Region    : eastus
      2021-08-28 00:15:47,446     RG        : rg_firenet-test_eastus
      2021-08-28 00:15:47,446 
      2021-08-28 00:15:47,446 ---------------------------------------------
      2021-08-28 00:15:47,446 
      2021-08-28 00:15:48,959 - Discover peerings
      2021-08-28 00:15:48,959   **Alert** vnet peering found in vn_firenet-test_VPC1-US-East
      2021-08-28 00:15:48,960   vpc1-to-vpc2 vn_firenet-test_VPC2-US-East (2292eeae-0602-442c-8635-1db589473ec9/rg_firenet-test_eastus)
      2021-08-28 00:15:48,960 - Discover VNG
      2021-08-28 00:15:48,960   VNG not found
      2021-08-28 00:15:48,961 - Discover subnet: 6 subnet(s)
      2021-08-28 00:15:48,961   Found Azure GatewaySubnet (rg_firenet-test_eastus) -- skipped 
      2021-08-28 00:15:48,961   sn_firenet-test_VPC1-US-East.subnet1 (rg_firenet-test_eastus)
      2021-08-28 00:15:48,961     address_prefix: 10.1.1.0/24
      2021-08-28 00:15:48,961     network_security_group: None
      2021-08-28 00:15:48,961     route_table: rt_firenet-test_VPC1-US-East.subnet1 (rg_firenet-test_eastus)
      2021-08-28 00:15:48,962     delegations: []
      2021-08-28 00:15:48,962   sn_firenet-test_VPC1-US-East.subnet2 (rg_firenet-test_eastus)
      2021-08-28 00:15:48,962     address_prefix: 10.1.2.0/24
      2021-08-28 00:15:48,962     network_security_group: None
      2021-08-28 00:15:48,962     route_table: rt_firenet-test_VPC1-US-East.subnet2 (rg_firenet-test_eastus)
      2021-08-28 00:15:48,962     delegations: []
      2021-08-28 00:15:48,962   default (rg_firenet-test_eastus)
      2021-08-28 00:15:48,962     address_prefix: 10.1.0.0/24
      2021-08-28 00:15:48,962     network_security_group: None
      2021-08-28 00:15:48,962     route_table: rt_firenet-test_VPC1-US-East.subnet1 (rg_firenet-test_eastus)
      2021-08-28 00:15:48,962     delegations: []
      2021-08-28 00:15:48,962   subnet-no-table2 (rg_firenet-test_eastus)
      2021-08-28 00:15:48,962     address_prefix: 10.1.4.0/24
      2021-08-28 00:15:48,962     network_security_group: None
      2021-08-28 00:15:48,962     route_table: vn_firenet-test_VPC1-US-East-dummy_rt (rg_firenet-test_eastus)
      2021-08-28 00:15:48,962     delegations: []
      2021-08-28 00:15:48,962   subnet-no-table1 (rg_firenet-test_eastus)
      2021-08-28 00:15:48,963     address_prefix: 10.1.3.0/24
      2021-08-28 00:15:48,963     network_security_group: None
      2021-08-28 00:15:48,963     route_table: vn_firenet-test_VPC1-US-East-dummy_rt (rg_firenet-test_eastus)
      2021-08-28 00:15:48,963     delegations: []
      2021-08-28 00:15:49,699 
      2021-08-28 00:15:49,699 ---------------------------------------------
      2021-08-28 00:15:49,699 
      2021-08-28 00:15:49,700     Route Tables
      2021-08-28 00:15:49,700 
      2021-08-28 00:15:49,700 ---------------------------------------------
      2021-08-28 00:15:49,700 
      2021-08-28 00:15:49,700 - Discover route table(s):
      2021-08-28 00:15:49,700   route table: rt_firenet-test_VPC1-US-East.subnet1
      2021-08-28 00:15:49,700     subnet: vn_firenet-test_VPC1-US-East/sn_firenet-test_VPC1-US-East.subnet1
      2021-08-28 00:15:49,700     subnet: vn_firenet-test_VPC1-US-East/default
      2021-08-28 00:15:49,700     **Alert** empty UDR route table rt_firenet-test_VPC1-US-East.subnet1
      2021-08-28 00:15:49,700   route table: rt_firenet-test_VPC1-US-East.subnet2
      2021-08-28 00:15:49,700     subnet: vn_firenet-test_VPC1-US-East/sn_firenet-test_VPC1-US-East.subnet2
      2021-08-28 00:15:49,700     **Alert** empty UDR route table rt_firenet-test_VPC1-US-East.subnet2
      2021-08-28 00:15:49,700   skip aviatrix created default vn_firenet-test_VPC1-US-East-dummy_rt
      
- The beginning of each **dm.arm.discovery** or **dm.arm.switch_traffic** run is marked by a line of number-sign characters (#), signifying the command and option that were used for the run.  In addition, one can identify the starting point of the latest run in the log by going to the end of the log file and search backward for the number-sign character. Similar structure applies to **dm.alert.log** as well.

- The logs file can be found at <terraform_output>/log.  They are also uploaded to the S3 bucket in <bucket>/dm/<spoke_account>/tmp at
the end of discovery or switch_traffic execution.

## S3 bucket

The S3 attributes in YAML specifies the bucket to be used in S3 for storing the logs and the  terraform files of each spoke VNET account.  If a new bucket is specified, **Discovery** will create the bucket with versioning and full privacy enabled.  If this is an existing bucket, **Discovery** will check if the bucket has versioning and full privacy enabled and will alert and terminate immediately if either one of the settings is missing.

- Discovery will backup the content of account folder into S3 at the end of each run when using the flag **--s3backup**.  The account folder contains the terraform files that will be retrieved at staging and switch_traffic time.

- Switch_traffic starts by downloading the account folder so it can
store the new subnet-route-table association resources to the existing terraform file.
At the end, it will upload the latest of the account folder back to S3.  In --revert mode, similar sequence occurs: 1) Terraform files are downloaded for the given accounts. 2) Previously added subnet-route-table resources are removed. 3) Upload all the account files back to S3.


## Additional checks

- **vCPU quota**. Check existing vCPU quota because each spoke gateway  will requires at least four vCPUs.  To enable this check, your subscription must register the resource provider Microsoft.Capacity.  See https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/resource-providers-and-types#azure-portal for the steps to register a resource provider in Azure portal.


- **Vnet lock**. Both **switch traffic and cleanup** support an optional **delete_vnet_lock** boolean attribute. When set to True, the script will check and delete the vnet lock at the beginning of the script.  The vnet lock, if exists, will prevent route table addition or deletion to the vnet, thus causing some migration operations to fail. 

  To create or delete management locks, one must have access Microsoft. Authorization/* or Microsoft. Authorization/locks/* actions. Of the built-in roles, only Owner and User Access Administrator are granted those actions. See https://docs.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources?tabs=json#who-can-create-or-delete-locks

  Go under Subscription/Access control (IAM)/Role assignments to add Owner or User Access Administrator role to your application.


## Integrating with terraform cloud VCS workflow

- Setup at terraform cloud

  1. Obtain a terraform cloud account and create an organization

  2. Create a workspace and select VCS flow.

     a. You will need to create a repo at github (or other git cloud) and link it to the workspace (See
     https://www.terraform.io/cloud-docs/vcs#connecting-vcs-providers-to-terraform-cloud for details). This repo will be used to commit the whole content of the **terraform_output** folder, created by dm.discovery. Here is a sample content of the **terraform_output** folder:
     ```
     drwxr-xr-x   5 userabc  staff  160 May  7 08:42 module_azure_brownfield_spoke_vnet
     drwxr-xr-x   4 userabc  staff  128 May 11 16:04 log
     drwxr-xr-x  13 userabc  staff  416 May 11 21:09 77a0a90d-77aa-4ff1-a1fb-77d777732aab
     ```
     - Note that the numeric folder is referred to as the account folder; its name is the Azure subscription ID of the spoke VNET you are migrating.

     b. While configuring your VCS in terraform cloud, populate the **Terraform Working Directory** field with the name of the **account folder** generated by dm.discovery, so Terraform cloud knows where to run terraform apply.

     c. Configure the **Apply Method** to use **Manual apply**, instead of **Auto apply**. Every commit to the repo will trigger a PLAN-and-APPLY run.  In Manual apply mode, you have a
     chance to review, confirm or discard a run, via the UI.

  3. Define a variable set for storing Azure subscription credential and controller credential in  terraform cloud: TF_VAR_azure_client_secret, AVIATRIX_PASSWORD, and AVIATRIX_USERNAME. You can share this variable set to the workspace just created or to all workspaces.

- Setup at your local terminal

  1. Make sure you are running latest terraform. Pre 1.0.0 does not support terraform **cloud** block.

  2. Export the environment variables AVIATRIX_PASSWORD, AVIATRIX_USERNAME, and TF_VAR_azure_client_secret locally for resolving the aviatrix provider dependence.

  3. In discovery.yaml, comment out tf_controller_access for configuring terraform to read the controller credential via the environment variables.

  4. In discovery.yaml, define **tf_cloud:** under terraform, e.g.:

  ```
    terraform:
      tf_cloud:
        organization: "<your organization name>"
        workspace_name: "<your workspace name>"
  ```

  5. Open up your controller security group to allow terraform cloud access.

  6. Run **terraform login** locally and follow the authentication instruction to copy and store the return token.

- Start migration
  
  1. Run **dm.arm.discovery** with the discovery.yaml at the terminal
  2. Run **terraform init** at the terminal  
  3. Run **source ./terraform-import-subnets.sh** at the terminal.
  4. Commit the terraform_output folder into the VCS repo, which should trigger a PLAN-and-APPLY run at the terraform cloud. Confirm the APPLY to stage the resources.
  6. Run **dm.arm.switch_traffic** at the terminal
  7. Run **source ./terraform-import-associations.sh** at the terminal.
  8. Commit the terraform_output folder again, and it will trigger another PLAN-and-APPLY run. Confirm the APPLY.
  9. Run **dm.arm.cleanup**.

- Considerations

  1. In cases where docker is used to run discovery migration related commands, the above migration steps should be modified to include repo access for recovering the terraform output folder when needed.
  2. Use one workspace per spoke Azure subscription to store the terraform state; that is, the workspace will hold the terraform states of all migrated VNETs in this Azure account.
  3. Use multiple workspace for the same Azure account. A separate repo is required for each workspace.