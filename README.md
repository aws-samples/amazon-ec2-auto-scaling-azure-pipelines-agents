# Using Amazon EC2 Auto Scaling to Manage Azure Pipelines Agent Capacity

This repo hosts templates written for the AWS Blog Post "[Using Amazon EC2 Auto Scaling to Manage Azure Pipelines Agent Capacity](https://aws.amazon.com/blogs/modernizing-with-aws/using-ec2-auto-scaling-to-manage-azure-pipelines-capacity/)" published on the [Microsoft Workloads on AWS](https://aws.amazon.com/blogs/modernizing-with-aws/) blog channel. 

## Overview
This is a sample solution using that lets you deploy either Windows or Linux-based Azure Pipelines agents with an Auto Scaling group, corresponding to a single agent pool. The solution can be deployed in your AWS account. It creates an Auto Scaling group that will manage the Amazon EC2-based agents. There are two [AWS Systems Manager](https://aws.amazon.com/systems-manager/) [documents](https://docs.aws.amazon.com/systems-manager/latest/userguide/documents.html) provisioned that will respond to Auto Scaling group lifecycle events through the use of [Amazon EventBridge rules](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-rules.html). One runs on instance launch. It  installs both the latest .NET build tools and git, and registers it with Azure DevOps. The other document unregisters the agent, and then terminates it. These documents are configured both for Windows and Linux hosts, and will run the correct set of commands based on the OS of the agent being provisioned.

<p align="center">
  <img src="https://github.com/aws-samples/amazon-ec2-auto-scaling-azure-piplelines-agents/blob/main/AzurePipelinesAgentsDiagram.png">
</p>

### Notes

The provided automation document will configure the Azure Pipelines agents with the following software:

**Windows**
- Install [Git](https://git-scm.com/) using the [Chocolatey](https://community.chocolatey.org/) client
- Install [Visual Studio Build Tools](https://aka.ms/vs/17/release/vs_BuildTools.exe) with all workloads selected


The Windows installation has been tested on Windows Server 2022

**Linux**
- Install the [.NET 6.0](https://dotnet.microsoft.com/en-us/download/dotnet/6.0) SDK (required by the Azure Pipelines agent) using yum
- Install Git using yum

The Linux installation has been tested on Amazon Linux 2023. If you are using a Linux distribution that does not support yum, then you can alter the "AzurePipelineScaleOut" SSM document accordingly.


## Deployment
### CloudFormation
#### Prerequisites
- You will need to have an Azure Pipelines agent pool provisioned. The provisioned agents will be added to and removed from this pool. 
- You will need to supply three AWS Systems Manager Parameter Store parameters to store values used by this solution. You will need to use the SecureString parameter type for any information considered sensitive. You will need parameters that will contain.
    -	The URL used to download the latest version of the Azure DevOps agent installer.
    -	The Personal Access Token (PAT) for a user with permissions to register Azure Pipelines agents. You can review the documentation to learn [how to create a Personal Access Token](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops). This parameter is required to be of type SecureString.
    -	This is the AWS Systems Manager Parameter store parameter where you store the AMI ID. If you are using a custom AMI that you have created to launch instances, then this will be a parameter that you create to store the AMI ID. If you wish to use the standard Windows or Linux template, you can use one of the public parameters that is automatically updated with the latest version of the AMI. You can learn [more in the documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-public-parameters-ami.html) on how to find the right parameter name for the AMI you want to use. Examples of latest AWS AMIs:

        | Operating system | Parameter Name
        | :---: | ---
        | Amazon Linux 2023 | `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`
        | Windows 2022 | `/aws/service/ami-windows-latest/Windows_Server-2022-English-Full-Base`



-	An AWS Account that you have permissions to deploy this into.
-	An Amazon VPC provisioned where your agents will be launched. The subnets that you specify will need internet connectivity. So if you intend to use private subnets, then ensure that there is a [NAT Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) provisioned.


#### Walkthrough

Start with downloading the [AzurePipelinesAgents.yaml](https://github.com/aws-samples/amazon-ec2-auto-scaling-azure-piplelines-agents/blob/main/Templates/CloudFormation/AzurePipelinesAgents.yaml) CloudFormation template, [create a stack on the AWS CloudFormation console](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html). The CloudFormation template will create the following resources:

#### Template Parameters
The stack template includes the following parameters:

##### **Azure DevOps Build Agent parameters** section
| Parameter | Required | Description
| --- | --- | ---
| AzureDevOpsAgentURLParameterName | Yes | This is the name of the String AWS Systems Manager Parameter Store parameter that contains the URL of the Azure DevOps agent software.
| AzureDevOpsPATParameterName | Yes | This is the name of the SecureString AWS Systems Manager Parameter Store parameter that will contain the Azure DevOps personal access token used for provisioning the agents.
| AzureDevOpsAgentPoolName  | Yes | The name of the agent pool these instances will be added to.
| AzureDevOpsOrganizationURL | Yes | The URL of the Azure DevOps organization that these agents will belong to.

##### **Instance Launch Template Parameters** section

| Parameter | Required | Description
| --- | --- | ---
| AmiIdParameterName | Yes | This is the AWS Systems Manager Parameter store parameter where the AMI to use is stored.
| InstanceType | Yes | This is the EC2 instance type that will be used for new agents. 
| Subnets | Yes | Choose the Amazon VPC subnets that new agents will be deployed into.
| Security group | Yes | The VPC Security group that will be attached to new agents when they are created.

##### **Scheduled scale in parameters** and **Scheduled scale out parameters** sections

| Parameter | Required | Description
| --- | --- | ---
| **ScheduledScaleInCron** and **ScheduledScaleoutCron** | Yes | [Cron expressions](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cron-expressions.html) that specify when the scale in and scale out actions will occur, respectively.
| **ScaleInDesiredCapacity** and **ScaleOutDesiredCapacity** | Yes | The desired capacity for the auto scaling group when in the scaled in and scaled out states, respectively.
| **ScaleInMinCapacity** and **ScaleOutMinCapacity** | Yes | The minimum capacity for the auto scaling group when in the scaled in and scaled out states, respectively.
| **ScaleInMaxCapacity** and **ScaleOutMaxCapacity** | Yes | The maximum capacity for the auto scaling group when in the scaled in and scaled out states, respectively.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
