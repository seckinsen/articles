# How to Deploy Serverless Stack on Alibaba Cloud Using Terraform

![terraform_alibaba_cloud](https://user-images.githubusercontent.com/17251181/68074417-d7025500-fdab-11e9-86e6-161e9f84ca53.jpg)

Nowadays, an approach to running `serverless` backend services is became popular. Here, the popular approach meaning is misnomer - we could briefly say that serverless is event-driven based cloud computing model which is needed no extra effort for maintenance of infrastructure. Cloud vendors such as Alibaba Cloud, AWS, Google Cloud, Azure etc. automatically manage the provisioning and allocation of computing resources.

With this approach, the building blocks of application is no longer a server but discrete chuck of code which is known as `function`. This event-driven based functions are invoked by particular actions which called as `trigger`. Consumers simply upload their functions to cloud vendor's infrastructure and trigger them when required. There is no need for scaling and load-balancing, the whole process is automatically computed by cloud vendor. We can trigger our functions once per month or million times per minutes and underlying infrastructure will handle incoming requests.

This guide is targeted at `Alibaba Cloud` and `Terraform` newbies and during guide I am going to introduce the basis of how to use Terraform to deploy our serverless stack on Alibaba Cloud. 

It assumes that you are familiar with the usual Terraform plan/apply/destroy workflow; if you're new to Terraform, the official Terraform [*documentation*](https://learn.hashicorp.com/terraform#getting-started "Terraform Getting Started Page") is good reference of introducing the basics of Terraform (i.e providers, resources, input variables, output variables, etc.). So in this guide, step by step we are going to build our infrastructure by putting those resources together.

1. **[Prerequisites](#prerequisites)**
2. **[Build Deployment Package](#build-deployment-package)**
3. **[Upload Deployment Package](#upload-deployment-package)**
4. **[View Function Execution Logs](#view-function-execution-logs)**
5. **[Arrange Function Permissions](#arrange-function-permissions)**
6. **[Create Function Compute Service](#create-fc-service)**
7. **[Create Function Compute Function](#create-fc-function)**
8. **[Create Function Compute Trigger](#create-fc-trigger)**
9. **[Deploy Serverless Stack](#deploy-serverless-stack)**
10. **[Clean up](#clean-up)**
11. **[Conclusion](#conclusion)**

You can find sample project at [*github*](https://github.com/seckinsen/terraform-function-compute-demo "Demo Project").

## <a name="prerequisites">Prerequisites</a>

While following this guide we will need an Alicloud account and Terraform installed working environment. If you do not already have an account, Alicloud provides free trial for expanding your cloud capabilities. You can sign up and get free account through [*link*](https://www.alibabacloud.com/campaign/free-trial "Alibaba Cloud Free Trial Page").

Event-driven based computing service on Alicloud is known as `Function Compute`. With free trial option we have monthly free charge of function execution chance for trying Alicloud Function Compute service. Also free function execution limit is automatically reset at the beginning of each month and it has no expiration date.

To prepare Terraform installed working environment, the official Terraform [*documentation*](https://learn.hashicorp.com/terraform/getting-started/install "Terraform Installing Page") can be followed or [*brew*](https://brewinstall.org/install-terraform-on-mac-with-brew/ "Terraform Brew Installation Guide") can be used for macOS users.

## <a name="build-deployment-package">Build Deployment Package</a>

The Function Compute expects function's implementation to be provided as an archive which contains function source code with any other 3rd party dependencies for function execution.

In this guide, our main objective is to deploy our serverless stack on Alicloud. The deployment step will be handled by Terraform. Terraform is written with `HCL` (Hashicorp Configuration Language) file. That's why firstly, we need to create our HCL file and define our infrastructure with code. Using this HCL file, Terraform will figure our deployment out.

Terraform is not a build tool, so the zip file which is our deployment file must be prepared using a separate build process prior to deploy it with Terraform. For building step, we have used one of the Terraform provider which is [*external*](https://www.terraform.io/docs/providers/external/index.html "Terraform External Provider Page") for building and archiving project binaries to produce the deployment package as a build artifact before deployment.

First step to use Terraform is to configure provider(s) which we need. To declare provider, following code block needs to be added in our HCL file.

	provider "external" {}

After declaring provider, following code block needs to be appended in our existing HCL file to run external scripts which aims to produce the deployment package. 

> For more detail about external scripts, please check the sample project at [*github*](https://github.com/seckinsen/terraform-function-compute-demo "Demo Project").

	data "external" "build" {
  		program = [
    		"./build.sh"
  		]
	}

	data "external" "archive" {
  		program = [
    		"./archive.sh",
    		"demo.zip",
    		"../src/"
  		]

  		depends_on = [
    		data.external.build
  		]
	}

> In the above code block, there is invisible dependency which declared by `depends_on` argument between resources. The `depends_on` argument is accepted by any resource or list of resources to create explicit dependencies. Terraform uses this dependency information to determine the correct order for execution plan. With the code block above, Terraform knows that the `build` operation must be handled before the `archive` operation. More information about `Resource Dependencies` you can check the official Terraform [*documentation*](https://learn.hashicorp.com/terraform/getting-started/dependencies.html "Terraform Resource Dependencies Page").

## <a name="upload-deployment-package">Upload Deployment Package</a>

After building deployment package, we need to upload zip file to storage service which is known as `Object Storage Service` on Alicloud for accessing deployment package by Function Compute service. To configure Alicloud resources by using Terraform, we need to declare Alicloud [*provider*](https://www.terraform.io/docs/providers/alicloud/index.html "Terraform Alibaba Cloud Provider Page"). That's why following code block needs to be appended in our existing HCL file.

	provider "alicloud" {
  		access_key = var.access_key
	  	secret_key = var.secret_key
  		account_id = var.account_id
  		region = var.region
	}
	
> As you can see with above code block, we need account informations to configure Alicloud resources with Terraform. You can read official Alicloud [*documentation*](https://partners-intl.aliyun.com/help/doc-detail/122151.htm "Alibaba Cloud Security Settings Overview Page") `Access Key` section for getting more information about security settings.

To create Object Storage Service on Alicloud, following code block needs to be appended in our existing HCL file

	resource "alicloud_oss_bucket" "demo" {
  		bucket = "demo"

  		depends_on = [
    		data.external.archive
  		]
	}

	resource "alicloud_oss_bucket_object" "demo" {
  		bucket = alicloud_oss_bucket.demo.bucket
  		key = "function/demo.zip"
  		source = "demo.zip"
	}

`alicloud_oss_bucket` resource will create an Object Storage Service bucket which name is declared with `bucket` argument and bucket name needs to be unique. `alicloud_oss_bucket_object` resource will put our archive file in Object Storage Service bucket. Here `bucket` argument specifies name of the bucket, `key` argument specifies name of the object in the bucket and `source` argument specifies path to the source file being uploaded to the bucket.

> In the above code block, there are invisible and visible dependencies. With invisible dependency declaration, Terraform knows that the `alicloud_oss_bucket` resource must be created after the `archive` operation finish successfully. With visible dependency, Terraform knows that `alicloud_oss_bucket_object` resource must be created after `alicloud_oss_bucket` resource created successfully.


## <a name="view-function-execution-logs">View Function Execution Logs</a>

Viewing log is useful option to system debugging and fault analysis. Function Compute service stores functions log in `Log Service` which is real-time data logging service on Alicloud. At Function Compute service create step, `Log Project` and `Log Store` are defined as Function Compute service settings. Here `Log Project` is resource management unit in Log Service and is used to isolate and control resources. `Log Store` is a unit in Log Service to collect, store, and query the log data.

To create Log Project and Log Store on Alicloud, following code block needs to be appended in our existing HCL file.

	resource "alicloud_log_project" "demo" {
	  name = "demo-log-project"
	}
	
	resource "alicloud_log_store" "demo" {
	  project = alicloud_log_project.demo.name
	  name = "demo-log-store"
	}
	
`alicloud_log_project` resource will create a Log Project which name is declared with `name` argument and Log Project name needs to be unique. `alicloud_log_store` resource will create a Log Store. Here `project` argument specifies name of the Log Project and `name` argument specifies name of the Log Store.

> In the above code block, there is visible dependency. With visible dependency, Terraform knows that `alicloud_log_store ` resource must be created after `alicloud_log_project` resource created successfully.

## <a name="arrange-function-permissions">Arrange Function Permissions</a>

The `Resource Access Management [RAM]` is a service provided by Alicloud for controlling resource access through permission levels. At Function Compute service create step, Resource Access Management role `Alibaba Cloud Resource Name [ARN]` attached to the Function Compute service to govern both who/what can invoke our function, as well as which resources our function has access to consume.

In our architecture, we need a role which have `Log Service` and `Elastic Network Interface [ENI]` permissions. Our Function Compute service needs Log Service access permission to store functions log and needs Elastic Network Interface access permission to access our custom `Virtual Private Cloud [VPC]`.

To provide required role, following code block needs to be appended in our existing HCL file.

	resource "alicloud_ram_role" "demo" {
	  name = "demo-role"
	
	  document = <<EOF
	  {
	    "Statement": [
	      {
	        "Action": "sts:AssumeRole",
	        "Effect": "Allow",
	        "Principal": {
	          "Service": [
	            "log.aliyuncs.com",
	            "fc.aliyuncs.com"
	          ]
	        }
	      }
	    ],
	    "Version": "1"
	  }
	  EOF
	
	  force = true
	}
	
	resource "alicloud_ram_role_policy_attachment" "demo_log_full_access" {
	  role_name = alicloud_ram_role.demo.name
	  policy_name = "AliyunLogFullAccess"
	  policy_type = "System"
	}
	
	resource "alicloud_ram_role_policy_attachment" "demo_ecs_network_interface_management_access" {
	  role_name = alicloud_ram_role.demo.name
	  policy_name = "AliyunECSNetworkInterfaceManagementAccess"
	  policy_type = "System"
	}

`alicloud_ram_role` resource will create a Resource Access Management role. Here `name` argument specifies name of the role, `document` argument specifies authorization strategy of the role and `force` argument specifies resource destroy strategy. `alicloud_ram_role_policy_attachment` resource will provide a role attachment. Here `role_name` specifies name of the role, `policy_name` specifies name of the role policy and `policy_type` which can be `Custom` or `System` specifies type of the role policy.

> In the above code block, there is visible dependency. With visible dependency, Terraform knows that `alicloud_ram_role_policy_attachment` resource must be attached after `alicloud_ram_role` resource created successfully.

## <a name="create-fc-service">Create Function Compute Service</a>

The Function Compute service is unit for managing and organizing Function Compute resources which is the base of launching function and trigger configuration. All functions belonging to a service share some common settings, such as log and VPC configurations, RAM role and internet access etc. Before we create function we need a created Function Compute service.

To create Function Compute service on Alicloud, following code block needs to be appended in our existing HCL file.

	resource "alicloud_fc_service" "demo" {
	  name = "demo"
	  description = "demo FC service"
	  log_config {
	    project = alicloud_log_project.demo.name
	    logstore = alicloud_log_store.demo.name
	  }
	  internet_access = false
	  role = alicloud_ram_role.demo.arn
	  vpc_config {
	    security_group_id = "SECURITY_GROUP_ID"
	    vswitch_ids = [
			"VSWITHC_ID"
	    ]
	  }
	}

`alicloud_fc_service` resource will create a Function Compute service. Here `name` argument specifies service name, `description` argument specifies service brief explanation, `log_config` argument specifies service log settings, `internet_access` argument specifies internet access switch of functions, `role` argument specifies RAM role ARN attached to service and `vpc_config` argument specifies service custom VPC access settings.

> In the above code block, there are visible dependencies in the above code block. With visible dependencies, Terraform knows that `alicloud_fc_service` resource must be created after `alicloud_log_project`, `alicloud_log_store` and `alicloud_ram_role` resources created successfully.

## <a name="create-fc-function">Create Function Compute Function</a>

After Function Compute service created successfully, we can create functions within the created Function Compute service. The Function Compute function is segment of application code that runs specific business logic during invocation. Function allows us to trigger execution of code in response to events in Alicloud.

To create Function Compute function on Alicloud, following code block needs to be appended in our HCL file.

	resource "alicloud_fc_function" "demo" {
	  service = alicloud_fc_service.demo.name
	  name = "demo"
	  description = "sample demo function"
	  memory_size = "128"
	  runtime = "nodejs8"
	  handler = "handler.demo"
	  oss_bucket = alicloud_oss_bucket.demo.bucket
	  oss_key = alicloud_oss_bucket_object.demo.key
	  timeout = 15
	  environment_variables = {
	  }
	}

`alicloud_fc_function` resource will create a Function Compute function. Here `service` arguments specifies associated Function Compute service name, `name` argument specifies function name, `description` argument specifies function brief explanation, `memory_size` argument specifies amount of memory which limits between 128 and 3072 in MB for our function can use at runtime, `runtime` argument specifies Function Compute [supported runtime types](https://www.alibabacloud.com/help/doc-detail/74712.htm "Function Compute Supported Runtimes"), `handler` argument specifies entry point of our code, `oss_bucket` argument specifies the Object Storage Service bucket location containing the function's deployment package, `oss_key` argument specifies the Object Storage Service key of an object containing the function's deployment package, `timeout` argument specifies the amount of time our function has to run in seconds, `environment_variables` arguments specifies defined environment variables for our the function.

> In the above code block, there are visible dependencies. With visible dependencies, Terraform knows that `alicloud_fc_function` resource must be created after `alicloud_oss_bucket` and `alicloud_oss_bucket_object` resources created successfully.

## <a name="create-fc-trigger">Create Function Compute Trigger</a>

Function Compute functions are generally invoked automatically by triggers, and these can take many [forms](https://www.alibabacloud.com/help/doc-detail/74707.htm "Function Compute Triggers"). To trigger our functions over http request with a unique URL, http trigger needs to be created.

To create Function Compute http trigger on Alicloud, following code block needs to be appended in our existing HCL file.

	resource "alicloud_fc_trigger" "demo" {
	  service = alicloud_fc_service.demo.name
	  function = alicloud_fc_function.demo.name
	  name = "demo-function-http-trigger"
	  role = alicloud_ram_role.demo.arn
	  type = "http"
	  config = <<EOF
	    {
	        "authType": "anonymous",
	        "methods" : ["GET"]
	    }
	  EOF
	}

`alicloud_fc_trigger` resource will create a Function Compute function. Here `service` arguments specifies associated Function Compute service name, `function` arguments specifies associated Function Compute function name, `name` argument specifies trigger name, `role` argument specifies RAM role ARN attached to trigger, `type` argument specifies type of trigger and `config` argument specifies trigger settings.

> In the above code block, there are visible dependencies. With visible dependencies, Terraform knows that `alicloud_fc_trigger ` resource must be created after `alicloud_fc_service`, `alicloud_fc_function` and `alicloud_ram_role` resources created successfully.

## <a name="deploy-serverless-stack">Deploy Serverless Stack</a>

Each Terraform provider can fulfill own functionality with its binary, but the binaries do not come with when we declare providers on HCL file. So when first starting to use Terraform, we need to run `terraform init` command to tell Terraform to scan the HCL file, figure out what providers we need, and download binaries of provider.

	$ terraform init
	
	Initializing the backend...
	
	Initializing provider plugins...
	- Checking for available provider plugins...
	- Downloading plugin for provider "external" (hashicorp/external) 1.2.0...
	- Downloading plugin for provider "alicloud" (hashicorp/alicloud) 1.58.1...
	
	(...)
	Terraform has been successfully initialized!


After required binaries downloaded, we can deploy our infrastructure with running `terraform apply` command:

	$ terraform apply
	(...)
	
	Terraform will perform the following actions:
	(...)
	
	Plan: 10 to add, 0 to change, 0 to destroy.
	
	Do you want to perform these actions?
	  Terraform will perform the actions described above.
	  Only 'yes' will be accepted to approve.
	Enter a value:
	
	
	Enter a value: yes
	
	(...)
	Apply complete! Resources: 10 added, 0 changed, 0 destroyed.

Congrats, we have just deployed our serverless stack with Terraform!


## <a name="clean-up">Clean Up</a>

Once we are finished with this guide, we can destroy our serverless stack with Terraform easily. All we need to do is to run the `terraform destroy` command.

	$ terraform destroy
	(...)
	 
	Terraform will perform the following actions:
	(...)
	
	Plan: 0 to add, 0 to change, 10 to destroy.
	Do you really want to destroy all resources?
	  Terraform will destroy all your managed infrastructure, as shown 
	  above. There is no undo. Only 'yes' will be accepted to confirm.
	  Enter a value:
	
	Enter a value: yes
	
	(...)
	Destroy complete! Resources: 10 destroyed.

## <a name="conclusion">Conclusion</a>

That's it. A complete picture of what it takes to deploy your serverless stack on Alicloud using Terraform. 

I hope I have been able to shown that Terraform is great `Infrastructure as Code [IAC]` tooling which has everything you need to deploy your serverless stack on Alicloud. This detailed post gives you a complete and viable approach for deploying Function Compute effectively. Hopefully this post gets your deployment steps faster.

Thanks for reading. If you have any questions or feedback, then please drop a note :)

> Also you can check my presentation through [*link*](https://slides.com/seckinsen/serverless-at-trendyol/fullscreen) for getting more information about Alicloud Function Compute service, limits and pricing.