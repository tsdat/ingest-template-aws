# **Tsdat AWS Pipeline Template**

This project template contains source code and supporting files for running multiple Tsdat pipelines via a single
serverless (i.e., lambda) application that you can deploy with the AWS SAM CLI. It includes the following files and folders:

- **lambda_function/** - Code for the application's Lambda function and Project Dockerfile.  The lambda function can run
multiple pipelines as defined under the pipelines subfolder.  Each pipeline's code is contained in a further 
subfolder (e.g., lambda_function/pipelines/a2e_buoy_ingest).  You will need to create a folder for each pipeline that
you would like to run via this lambda template. The lambda_function/pipelines/runner.py class contains a map of
regular expressions which are used to map file names of a certain pattern to the pipeline and config files
that will be used to process that specific file.  The regular expressions will need to be updated to work with your
input file name patterns.
- **data/** - Sample data to use for running tests local
- **tests/** - Unit tests for the application code. **`tests/test_pipeline.py`** is used to run unit tests on your local filesystem.
Anyone can run the local filesystem tests.  **`tests/test_lambda_function.py`** is used to run lambda function unit tests against test
input/output S3 buckets.  You will need an AWS account and permissions to read from the input bucket and write to the output bucket in order to run the AWS unit tests.
- **template.yaml** - A template that defines the application's AWS resources that will be created/managed in a single
software stack via AWS Cloud Formation.  This template defines several AWS resources including Lambda functions,
input/output S3 buckets, and event triggers.  You should modify this template to create the appropriate resources 
needed for your AWS environment.
- **samconfig.toml** - A config file containing variables specific to the deployment such as input/output bucket names,
stack name, and the logging level to use.  This file is automatically generated/updated when you deploy your pipeline to AWS using 
`deploy --guided` (see below).

First follow the steps in the following section, **Setting up your pipeline(s) for local development**, in order to get started and ensure that your pipeline can run locally.  Then a user with AWS administrator permissions for your account can follow the steps in **"Deploying your pipeline(s) to AWS"** in order to deploy and test your pipeline on AWS.

# **Setting up your pipeline(s) for local development**
Anyone can contribute to your pipeline code and run local tests without requiring an Amazon account.  Follow these
steps first to set up one or more pipelines and test them locally to ensure they are working properly.

## **Prerequisites**
You will need Python for local testing and development - [Python 3 installed](https://www.python.org/downloads/)
You will also need Tsdat:
```bash
pip install tsdat
```

## **Configuring your pipeline(s)**
You can set up one or more pipeline to run via this template.  Please ensure that you have a unique name to identify each pipeline.  We will refer to this as the **PIPELINE_NAME** throughout the rest of this document.  Also ensure that you have a unique name to identify each location where your pipeline will run.  We will refer to this as the **LOCATION** throughout the rest of this document.

**For each pipeline you will deploy**, do the following:

### **1) Set up your pipeline data folder**
This is the folder that contains local data for testing.  Under the **data/** folder, create a folder with your PIPELINE_NAME.  Then under the PIPELINE_NAME folder, create folders for each LOCATION.  Under each LOCATION folder, deposit test files that are appropriate for that pipeline and location.

### **2) Configure regular expressions in your pipeline runner**
The pipeline runner is located at **`lambda_function/pipelines/runner.py`** and is used to control which pipeline and location should be run for a given input file.  This is done by using regular expresions to map file name patterns to pipelines and locations.  At the top of the runner.py file, please change the
pipeline_map and location_map to match the file patterns for your different
pipelines.  The keys for the pipeline_map should be your PIPELINE_NAME.  The keys for the location_map should be your LOCATION_NAME(s).

```python

pipeline_map = {
    'PIPELINE_NAME': re.compile('FILE_REGEX_PATTERN')
}

location_map = {
    'LOCATION_NAME': re.compile('FILE_REGEX_PATTERN')
}
```

Example 1: 

```python
# For data from multiple 'locations':

pipeline_map = {
    # Specify general file naming convention
    'a2e_buoy_ingest': re.compile('buoy\\..*\\.(?:csv|zip|tar|tar\\.gz)')
}

location_map = {
    # Specify local file attribute
    'humboldt': re.compile('.*\\.z05\\..*'),
    'morro'   : re.compile('.*\\.z06\\..*')
}
```

Example 2:

```python
# For data from one 'location':
# (input filename format here is 'tide_YYYYDDMMhhmmss.log')

pipeline_map = {
    # General regex file structure
    'tide_data_ingest': re.compile('tide_\d{14}.log'),
}

location_map = {
    # Local can be same as general
    'mcrl': re.compile('tide_\d{14}.log'),
}
```

### **3) Create your pipeline code and configuration files**

* **Rename your pipeline code folder**

  Under the **`lambda_function/pipelines`** folder, rename the a2e_buoy_ingest to your PIPELINE_NAME. *(Make and rename a copy of the a2e_buoy_ingest folder for every pipeline being deployed.)*

* **Rename your config files**

  For each pipeline LOCATION, create a pipeline configuration file under the **lambda_function/pipelines/PIPELINE_NAME/config** folder.

  ```bash
  pipeline_config_LOCATION.yml
  ```

  For example
  ```bash
  pipeline_config_humboldt.yml
  ```

  You can rename the existing files to match your LOCATION(s) or you can create a new file as needed.

  For more information on tsdat's configuration files see our documentation on customizing [Configuration Files](https://tsdat.readthedocs.io/en/latest/configuring_tsdat.html#configuration-files)

* **Configure your pipeline's file handlers**

  Update the **`lambda_function/pipelines/config/storage_config.yml`** file to specify the file handlers required for your pipeline. 

* **Configure your pipeline's code**

  Edit the **`lambda_function/pipelines/PIPELINE_NAME/pipeline.py`** file with the correct code for your ingest pipeline.

  For more information on code customization you can apply in tsdat, please see our documentation on [Code Customizations](https://tsdat.readthedocs.io/en/latest/configuring_tsdat.html#code-customizations)


* **Configure your pipeline's file handlers**

  If needed, add custom `filehandlers.py` to the **`lambda_function/pipelines/PIPELINE_NAME/`** folder. See the `tsdat/examples/` folder for example custom file handlers or `tsdat-template-local/pipeline/filehanders.py` for a dummy file handler template.

  Update the **`lambda_function/pipelines/config/storage_config.yml`** file to specify the file handlers required for your pipeline.
  
    ```bash
    file_handlers:
      inputs:
        <ext>:
          file_pattern: '.*\.<ext>'
          classname: pipelines.PIPELINE_NAME.filehandlers.<CustomFileHandler>
  ```
  For example:

    ```bash
    file_handlers:
      input:
        csv:
          file_pattern: '.*\.csv'
          classname: tsdat.io.filehandlers.CsvHandler
    ```


## **Configuring your unit tests**
Edit the **`tests/test_pipeline.py`** file to define unit tests for your PIPELINE and LOCATION.

Specifically, copy/rename the **test_buoy** method and edit the code accordingly to run one test for each LOCATION:

```python
    def test_PIPELINE_NAME(self):
        run_pipeline([os.path.join(data_dir, 'PIPELINE_NAME/LOCATION_NAME/DATA_FILE_NAME')])
```
For example:

```python
    def test_a2e_buoy_ingest(self):
        run_pipeline([os.path.join(data_dir, 'a2e_buoy_ingest/humboldt/buoy.z05.00.20201201.000000.zip')])
        run_pipeline([os.path.join(data_dir, 'a2e_buoy_ingest/morro/buoy.z06.00.20201201.000000.zip')])
```

## **Testing your pipeline locally**
```bash
$ python tests/test_pipeline.py
```
**Please follow the Code Tour located at XXXX to learn how to interactively debug your pipeline with VSCode.**
TODO: XXXX
.

# **Deploying your pipeline(s) to AWS**
Once you have configured and tested your pipeline locally, you are ready to deploy it to AWS.  This section describes the steps needed to build, deploy, and test your pipeline via an AWS lambda serverless function.

## **Prerequisites**
The following are required in order to deploy your pipeline to AWS:

### **1) Create an AWS account for your project**
This is necessary for admins who will deploy your application to AWS.  Each institution may have different policies for
creating new project accounts on AWS.  Please contact your local AWS administrator for assistance.

### **2) Create an AWS account for your user with the Adminstrator role**
Any user who will use this template to deploy resources to AWS must have an AWS account with the **Administrator** role for your project. Please contact your local AWS administrator for assistance.

### **3) Create an Amazon Elastic Container Registry (ECR) repository**
[Create an ECR repository](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html) where your lambda
Docker image will be deployed.

### **4) Install Docker**
Required to build the AWS Lambda image. [Install Docker community edition](https://hub.docker.com/search/?type=edition&offering=community)

### **5) Install AWS CLI**
Required to deploy the software stack defined in template.yaml.  [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

### **6) Install AWS SAM CLI**
Required to deploy the software stack defined in template.yaml
The Serverless Application Model Command Line Interface (SAM CLI) is an extension of the AWS CLI that
 adds functionality for building and testing Lambda applications. It uses Docker to run your functions in an Amazon
 Linux environment that matches Lambda. It can also emulate your application's build environment and API.

[Install the SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)

### **7) Configure AWS authentication for using the CLI**
You will need to [configure your CLI AWS authentication](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) so that the CLI can properly authenticate commands to AWS.  **NOTE:  There are multiple ways to configure your CLI permissions.**  Since the mechanism depends upon how your institution has configured authentication to AWS,
please contact your AWS adminstrator for assistance. 

### **8) Set your AWS_PROFILE environment variable (optional)**
If you configured your CLI authentication to use SSO (single sign-on), then
you must set your **AWS_PROFILE** environment variable to point to the profile name you selected in Step 7 above.  For example, from a bash shell:

```bash
export AWS_PROFILE='my-profile-name'
```
We recommend you set this variable in your .bashrc file so that you don't have to run this command every time you open a new terminal to execute CLI commands.

## **Configuring your AWS resource stack**
This project will deploy an AWS resource stack defined by the **`template.yaml`** file.  The following resources will be created:

1. A lambda function to run your pipelines
2. An input S3 bucket
3. An output S3 bucket
4. The required permissions for your lambda function to access the input and output buckets

You may optionally edit the **`template.yaml`** file to configure default parameters for your stack.  Specifically, you only need to modify the following parameters if you would like to change the default values.

```yaml
Parameters:
  # The source/input bucket where raw files will be placed
  SourceBucketName:
    Type: String
    Default: "tsdat-pipeline-inputs"  #<==== Change this if you want your input bucket to be named differently

  # The destination bucket where tsdat pipeline outputs will be placed
  DestinationBucketName:
    Type: String
    Default: "tsdat-pipeline-outputs"  #<==== Change this if you want your output bucket to be named differently

  LambdaFunctionName:
    Type: String
    Default: "tsdat-pipeline-lambda"   #<==== Change this if you want your lambda function to be named differently

  # The logging level to use
  LoggingLevel:
    Type: String
    Default: INFO
```

## **Building your lambda function**
Once you have configured your stack parameters, use the `sam build` command to build your lambda function for deployment.

```bash
$ sam build
```

The SAM CLI builds a docker image from a Dockerfile, copies the source of your application inside the Docker image,
and then installs dependencies defined in `lambda_function/requirements.txt` 
inside the docker image. The processed template file is saved in the `.aws-sam/build` folder.


## **Deploying your AWS resource stack for the first time**
To deploy your AWS application stack for the first time, run the following in your shell:

```bash
sam deploy --guided
```

This will package and deploy your application to AWS, with a series of prompts asking for different parameter values.
The default values are shown in brackets [].  If you just hit Enter without typing a different value, the default
value will be used.  All of the default values will work out of the box except the Image Repository.  **You must paste
the URI for the ECR image repository created in Prerequisite #3 above**.  Each of the prompts are decribed in more detail below:

* **Stack Name**: The name of the stack to deploy to CloudFormation. This should be unique to your account and region, 
and a good starting point would be something matching your project name.
* **AWS Region**: The AWS region you want to deploy your app to.
* **Source Bucket**: The name of the S3 bucket where incoming raw files will be placed to trigger the pipeline.  Note
that this stack deployment WILL CREATE THIS BUCKET FOR YOU, so **the name must not already be in use**.
* **DestinationBucket**: The name of a different S3 bucket where the outputs of the tdat pipeline will be placed.   Note
that this stack deployment WILL CREATE THIS BUCKET FOR YOU, so the **name must not already be in use**.
* **LoggingLevel**: The logging level that will be used for python logging statements which get written to the lambda
function's CloudWatch logs.
* **Image Repository for LambdaFunction**: A URI for the ECR image repository that will be used to store your function's 
Docker image.  For example, `332883119153.dkr.ecr.us-west-2.amazonaws.com/a2e-tsdat-test`
* **Confirm changes before deploy**: If set to yes, any change sets will be shown to you before execution for manual review.
 If set to no, the AWS SAM CLI will automatically deploy application changes.
* **Allow SAM CLI IAM role creation**: Many AWS SAM templates, including this example, create AWS IAM roles required for the 
AWS Lambda function(s) included to access AWS services. By default, these are scoped down to minimum required permissions. 
To deploy an AWS CloudFormation stack which creates or modified IAM roles, the `CAPABILITY_IAM` value for `capabilities` must be provided. 
If permission isn't provided through this prompt, to deploy this example you must explicitly pass `--capabilities CAPABILITY_IAM` to 
the `sam deploy` command.
* **Save arguments to samconfig.toml**: If set to yes, your choices will be saved to a configuration file inside the project, so that in the future you can just re-run `sam deploy` without parameters to deploy changes to your application.

  An Example guided deployment is shown below:

  ```bash
  $ sam deploy --guided

  Configuring SAM deploy
  ======================

          Looking for config file [samconfig.toml] :  Found
          Reading default arguments  :  Success

          Setting default arguments for 'sam deploy'
          =========================================
          Stack Name [tsdat-pipeline-stack]:
          AWS Region [us-west-2]:
          Parameter SourceBucket [tsdat-pipeline-inputs]:
          Parameter DestinationBucket [tsdat-pipeline-outputs]:
          Parameter LoggingLevel [DEBUG]:
          Image Repository for LambdaFunction [332883119153.dkr.ecr.us-west-2.amazonaws.com/a2e-tsdat-test]:
            lambdafunction:python3.8-v1 to be pushed to 332883119153.dkr.ecr.us-west-2.amazonaws.com/a2e-tsdat-test:lambdafunction-4bdc10a8a50a-python3.8-v1

          #Shows you resources changes to be deployed and require a 'Y' to initiate deploy
          Confirm changes before deploy [Y/n]: Y
          #SAM needs permission to be able to create roles to connect to the resources in your template
          Allow SAM CLI IAM role creation [Y/n]: Y
          Save arguments to configuration file [Y/n]: Y
          SAM configuration file [samconfig.toml]:
          SAM configuration environment [default]:
  ```


## **Deploying your lambda function**
Once you have used `sam deploy --guided` to update your samconfig.toml file the first time, you can subsequently deploy with the `sam deploy` command.

```bash
$ sam deploy
```
This will package and deploy your built application template to AWS.


## **Using Your Pipeline on AWS**
Once your pipeline has been deployed, you can trigger it by simply placing raw files into the Source Bucket you specified in the template.yaml file.


## **Testing your Lambda Function Locally (no container)**

If you find bugs with your lambda function, it may be easier to test/debug your lambda function without involving the docker container. Follow these steps to run your lambda function locally outside a docker container.

* **Create test S3 buckets**

  Create a test input S3 bucket on AWS, referred to as TEST_INPUT_BUCKET. Your test input data will go here.  Upload your test files from the **data/** folder here.  **Remeber that the paths used in your S3 bucket key must exactly match the paths in your data folder (i.e., `a2e_buoy_ingest/humboldt/buoy.z05.00.20201201.000000.zip`)

  Create a test output S3 bucket on AWS, referred to as TEST_OUPUT_BUCKET.  The outputs of your tests will go here.

* **Configure your lambda test events**

  The **`tests/events`** folder contains simulated S3 bucket events that are used for running lambda_Function tests on your TEST_INPUT_BUCKET.  In order to run  `test_lambda_function.py`, you will need to update the events to match your AWS
  test data that you placed in your TEST_INPUT_BUCKET.  Specifically, you will need to update the bucket name property and object key property to match your specific test files as shown in the following
  snippet from an s3-event.json file.

  ```json
  {
        "s3": {
          "s3SchemaVersion": "1.0",
          "configurationId": "4586d195-b484-464c-97f8-027348232818",
          "bucket": {
            "name": "a2e-tsdat-test", <==== change this to match TEST_INPUT_BUCKET
            "ownerIdentity": {
              "principalId": "A21WSIO2BY6RXE"
            },
            "arn": "arn:aws:s3:::a2e-tsdat-test"
          },
          "object": {
            "key": "a2e_buoy_ingest/humboldt/buoy.z05.00.20201201.000000.zip", <==== change this to match the path to your test file
            "size": 3679233,
            "eTag": "84703366a2bac7da56749b6cdbe37de1",
            "sequencer": "00606E62D96731561E"
          }
        }
  }
  ```

* **Configure your unit tests**

  **`tests/test_lambda_function.py`** is used to run your lambda function unit tests.  Edit this file to define unit tests for your PIPELINE and LOCATION.

  Specifically, copy/rename the **test_buoy** method and edit the code accordingly to run one test for each LOCATION:

  ```python
    def test_PIPELINE_NAME(self):
        self.run_pipeline('PIPELINE_NAME', 'LOCATION')
  ```
  For example:

  ```python
    def test_a2e_buoy_ingest(self):
        self.run_pipeline('a2e_buoy_ingest', 'humboldt')
        self.run_pipeline('a2e_buoy_ingest', 'morro')
  ```

* **Run your lambda unit tests**

  ```bash
  $ python tests/test_lambda_function.py
  ```
.

## **Running Local Tests via Docker Container**
You may also want to test your pipeline locally by running inside the same lambda container that will be used on AWS.  To do this, you will need to have Docker installed on your machine.  Follow these steps to run your lambda container locally:

1. Configure your lambda unit tests as described in the previous section
2. Use SAM to build and run the pipeline locally:
    ```bash
    sam build 
    sam local invoke tsdat-pipeline-lambda --event tests/events/a2e_buoy_ingest/morro/s3-event.json 
    ```
.

## **Fetch, tail, and filter Lambda function CloudWatch logs**

To view/filter logs for your lambda function, you can use the
[Amazon CloudWatch](https://docs.aws.amazon.com/cloudwatch/index.html) user interface.

Alternatively, to simplify troubleshooting, SAM CLI has a command called `sam logs`. `sam logs` lets you 
fetch logs generated by your deployed Lambda function from the command line. In addition to printing the logs on the
 terminal, this command has several nifty features to help you quickly find errors.

Assuming you used the default stack name, to tail your pipeline logs, you can run this command:

```bash
$ sam logs -n LambdaFunction --stack-name tsdat-pipeline-stack --tail
```

You can find more information and examples about filtering Lambda function logs in the 
[SAM CLI Documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-logging.html).

.

## **Cleanup**

To undeploy your lambda function, use the AWS CLI. Assuming you used the default stack name provided by this template,
you can run the following:

```bash
aws cloudformation delete-stack --stack-name tsdat-pipeline-stack
```

**-----> NOTE: You MUST make sure that both your Source Bucket and Destination Bucket are completely empty or else you
will not be able to delete your stack.**

.


## **ADVANCED:  Further customizing your application stack**
This application template uses AWS Serverless Application Model (AWS SAM) to define application resources that will
be created automatically on AWS by the deploy command.
AWS SAM is an extension of AWS CloudFormation with a simpler syntax for configuring common serverless application 
resources such as functions, triggers, and APIs.
See the [AWS SAM developer guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html) for an 
introduction to SAM specification, the SAM CLI, and serverless application concepts.

By default, this template will create the following resources on AWS:
```bash
CloudFormation stack changeset
---------------------------------------------------------------------------------------------------------------------------------------------
Operation                           LogicalResourceId                   ResourceType                        Replacement
---------------------------------------------------------------------------------------------------------------------------------------------
+ Add                               DestinationS3Bucket                 AWS::S3::Bucket                     N/A
+ Add                               LambdaFunctionFileUploadedPermiss   AWS::Lambda::Permission             N/A
                                    ion
+ Add                               LambdaFunctionRole                  AWS::IAM::Role                      N/A
+ Add                               LambdaFunction                      AWS::Lambda::Function               N/A
+ Add                               SourceS3Bucket                      AWS::S3::Bucket                     N/A
---------------------------------------------------------------------------------------------------------------------------------------------
```

This should be adequate for most deployments, but in some cases you may need to change the configuration.  Here are a couple
of examples:

1. You need to trigger the pipeline from an AWS notification service message instead of from files being placed in the Source Bucket.
2. You need to trigger the pipeline from an existing S3 bucket that was not created by this template.

If you need to modify this template, it will require an in-depth knowledge of the [SAM specification](https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md),
and the [AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)
resource types.  Contact your local AWS expert for assistance.

