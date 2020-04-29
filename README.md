# aws-trend-micro-dssc

This guide details steps and procedures you can follow to create, launch, and implement you own standalone container scanning solution within the AWS ecosystem.  This approach uses a commercial product by Trend Micro called [Deep Security Smart Check](https://www.trendmicro.com/en_us/business/products/hybrid-cloud/smart-check-image-scanning.html) (DSSC) as a proof-of-concept and provides examples of how integrate with AWS CodePipeline.

## Pipeline Architecture

High level overview of the pipeline architecture.

![DSSC Pipeline High-Level Architecture](https://github.com/stelligent/aws-trend-micro-dssc/blob/master/docs/dssc_pipeline.png)

## AWS Services Used

- AWS CloudFormation
- AWS ECR
- AWS EKS
- AWS ApiGateway
- AWS Lambda
- AWS Parameter Store
- AWS CodePipeline
- AWS CodeBuild
- AWS CloudWatch

## Getting Started

This application requires a few steps that must be executed as specified.  Please refer to the [prerequisites](#Prerequisites) section prior to running any commands contained in this article to ensure you have the required packages and software installed.

__WARNING__: You may incur charges with the use of some of these AWS Services, specifically EKS if you leave your cluster up and running over time.

### Prerequisites

Ensure that the following are installed or configured on your workstation before deploying DSSC.

- Git
- Docker
- AWS CLI
- eksctl
- helm
- Make
- jq

### Installation

- Clone this [repository](https://github.com/stelligent/aws-trend-micro-dssc)
- Install and configure [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)
- Install and configure [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)
- Install and configure [helm](https://helm.sh/docs/intro/)

## Deployments

### Create EKS Cluster

DSSC requires a Kubernetes cluster to run on so we’ll be using AWS EKS.

```
make deploy-cluster
```

This can take upwards of 15 minutes to create.

This command uses eksctl to create the cluster and two unmanaged nodes in preparation for the DSSC deployment.

### Deploy DSSC

Deploy DSSC to the EKS cluster created previously.

```
make deploy-dssc
```

This command uses helm to deploy DSSC to the cluster.  Detailed instructions are located at https://github.com/deep-security/smartcheck-helm.

There will be instructions output to the console on where and how to login to the DSSC administrative UI.

__NOTE__: The DSSC package deploys with a self-signed cert for HTTPS.  You can choose to upload a valid cert, or simply ignore the error.  For this blog and simplicity reasons, we ignored the error in the browser and proceeded to the site.  Refer to Replace Self-Signed Certificate in the DSSC deployment guide.

### Change DSSC Password

DSSC requires you to update the password after installation.  You can’t execute any API calls until the password is changed, and our pipeline will be executing API calls.  Log into the UI and change the password (and don’t forget it!).  You will need the new password in the subsequent step.

![DSSC Change Password Screen](https://github.com/stelligent/aws-trend-micro-dssc/blob/master/docs/dssc_password_change.png)

### Create DSSC SSM Parameters

We have everything we need to connect to DSSC, so now it’s time to store those in AWS Parameter Store (SSM) so the pipeline can communicate with DSSC when it runs.

```
make deploy-dssc-ssm NEW_DSSC_PASSWORD=<the one you were supposed to remember>
```

This command will create four parameter store values all prefixed with `/pipeline/example/trendmicro/dssc` to store the URL, username, password, and secret to the DSSC API/UI.  The pipeline will use these values when it runs to initiate scans.

### Create ECR Repository

We need an image repository to store our sample application image.  We will be using AWS ECR.

```
make deploy-ecr
```

This command will create an AWS ECR image repository.  This repository is used to house the sample app image which DSSC will use when a scan is initiated.

### Push Sample Image

We need an image to scan so a sample has been provided.  To build and push it to the ECR repository we just created, run the following.

```
make build-and-push-docker-image
```

This will build a docker image using the application located in `./sample_app` and push it to the ECR repository created previously.

### Create Webhook

DSSC supports webhooks, which is a way for DSSC to communicate with other processes when certain events occur, like when a scan completes.  Leveraging a webhook is how we will integrate DSSC into our CI/CD pipeline. This means we need something for DSSC to call when scans are complete, so we will create a webhook for DSSC to call.

```
make deploy-webhook
```

This command creates an AWS ApiGateway, AWS Lambda, and an AWS SSM parameter storing the URL to access it.
The lambda code is in `cloudformation/webhook.yaml`.  This code takes the incoming JSON posted to it from DSSC and simply looks at the  `critical` and `high` errors counts in the JSON.  The code then approves or rejects the `Approve Deployment` pipeline action.  The code also checks the X-Scan-Event-Signature (using the DSSC Secret that was created with the deploy-dssc-ssm command) header to determine if the call is a valid one from DSSC.  A 401 is returned if the signature is invalid.  Refer to [Securing Web Hooks](https://github.com/deep-security/smartcheck-helm/wiki/Secure-web-hooks) for more information.

### Create Pipeline

Now we are ready to create the Pipeline.

```
make deploy-pipeline
```

This command creates an AWS CodePipeline pipeline and an AWS CloudWatch event rule.  The event rule will trigger the pipeline when new images are pushed to the ECR.

This pipeline has four stages
- Source: Triggered by a new image uploaded to ECR and tagged ‘latest’.
- Build: Calls DSSC API to initiate a new scan using the new image in ECR.
- Approve Deployment: A manual approval step that is automatically set to Approve/Rejected by the DSSC scan.
- Deployment: A mock deployment of the image.

The pipeline will trigger automatically upon creation.  Now move on to the Verification [#Verfication] section.


## Verification

### Pipeline

We have deployed everything and created our pipeline.  Now, let’s check the status.
This command displays the status of the different stages in the pipeline.

```
make get-pipeline-status
```

Console output:
```
=== Getting trend-mirco-dssc-pipeline Status ===
-------------------------------------
|         GetPipelineState          |
+--------------------+--------------+
|  Source            |  Succeeded   |
|  Build             |  InProgress  |
|  ApproveDeployment |  None      |
|  Deploy            |  None        |
+--------------------+--------------+
```

Once the pipeline stage `ApproveDeployment` is Succeeded or Failed run the following command to get a description of the results.

```
make get-pipeline-stage-result
```

Console output:

```
=== Getting trend-mirco-dssc-pipeline stage 'ApproveDeployment' Summary ===
There are 3 critical issues and 22 high issues detected. Full results available at https://blah.us-east-1.elb.amazonaws.com/scans/aaf52a3d-d759-434a-a1c3-1de696197212
```

### DSSC
You can also check the scan status in DSSC at any time.
This command accesses DSSC via the API and retrieves the scan status.

```
make get-scan-status
```

Console output:

```
=== Getting DSSC Scan Status ===
aaf52a3d-d759-434a-a1c3-1de696197212: in-progress
```

The output is `scan_id: status`.

The scan is complete when it shows a status of ‘complete-with-findings’ in this example.
You can view the full results by running the following:

```
make get-scan-result
```

### Adjust Failure Tolerance

Built into the Lambda code, is the ability to adjust the error tolerances for both critical and high errors.  We will now relax those settings in order to allow the pipeline to succeed and execute the deployment action.

```
make adjust-error-tolerance
```

### Retrigger Pipeline

Now that the error tolerances have been adjusted, retrigger the pipeline.

```
make retrigger-pipeline
```

Once the DSSC status `make get-scan-status` is showing ‘completed-with-findings’, rerun

```
make get-pipeline-status
```

Console output:

```
=== Getting trend-mirco-dssc-pipeline Status ===
-------------------------------------
|         GetPipelineState          |
+--------------------+--------------+
|  Source            |  Succeeded   |
|  Build             |  Succeeded   |
|  ApproveDeployment |  Succeeded   |
|  Deploy            |  InProgress  |
+--------------------+--------------+
```

To view the pipeline stage comments, once again run

```
make get-pipeline-stage-result
```

Console output:
```
=== Getting trend-mirco-dssc-pipeline stage 'ApproveDeployment' Summary ===
Full results available at https://blah.us-east-1.elb.amazonaws.com/scans/6526820b-f8e8-47ce-b528-9ed7319873b1
```

## Clean-Up

Remember to tear everything down to avoid excessive charges in your AWS account.

```
make teardown
```

## Contributing

None yet

## Versioning

None yet

## License

MIT Licencse Copyright (c) 2019 Mphasis-Stelligent, Inc. https://stelligent.com