
## Overview
This repo is an end-to-end template for training and deploying a container & creating and serving endpoints using 
Amazon SageMaker. 


This repository is a copy of [shapemaker](https://github.com/smaakage85/shapemaker), which is licensed under the MIT license.

The template has some modifications that allows the setup to work on M2 laptops. The following functionality is supported:

- a minimalistic template for model code
- a template for a docker image for model training
- an endpoint docker image template for real-time inference
- command-line functions for interacting with the model/endpoint
- command-line functions for delivering and integrating the model w/SageMaker
- workflows enabling continuous integration/delivery.

## Requirements

The setup was done on a Mac (M2), it might need to be tweaked slightly to accommodate other devices.

**Cloud Services**
- Amazon Web Services*
  - User and Policy: Create a new user 'vmascarenhas' and attach the following policies:
    ```
        AmazonSageMakerFullAccess
    ```
    ```
        {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": [
                    "ec2:DescribeNetworkInterfaces",
                    "ec2:DescribeVpcs",
                    "kms:ListAliases",
                    "iam:ListRoles",
                    "ec2:DescribeSubnets",
                    "ec2:DescribeSecurityGroups",
                    "ec2:CreateNetworkInterface",
                    "iam:CreatePolicy",
                    "ec2:DeleteNetworkInterface",
                    "iam:CreateRole",
                    "iam:AttachRolePolicy"
                ],
                "Resource": "*"
            }
        ]
        }
    ```
    
  - S3 bucket
    -  The S3 bucket acts as the location where you’ll store data for various ML processes, 
    including passing training and test data to the ML algorithms, temporary data and output 
    from the ML algorithms (e.g. model files). Be sure to create the S3 bucket in the same region 
    that you intend to create the Sagemaker instance.


**Software**
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  - ```
    curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
    sudo installer -pkg ./AWSCLIV2.pkg -target /   
    which aws
    aws --version
    ```
- [direnv](https://direnv.net/docs/installation.html) 
  - ```brew install direnv```
- [gettext](https://www.drupal.org/docs/8/modules/potion/how-to-install-setup-gettext)
  - ```brew install gettext```
- [docker](https://docs.docker.com/get-docker/)
- [docker compose](https://docs.docker.com/compose/install/)
- [pyenv](https://github.com/pyenv/pyenv)
  - ```
    brew install pyenv
    brew install pyenv-virtualenv
    
    pyenv install -v 3.11.3
    pyenv virtualenv 3.11.3 v3.11.3
    
    pyenv activate v3.11.3
    ```
- [python](https://www.python.org/downloads/)
- [Cookiecutter](https://pypi.org/project/cookiecutter/)
  - ```pip install cookiecutter```

**CI/CD**
- [Github Actions](https://github.com/features/actions)


## Usage

### Create project from template
Create a project template using `cookiecutter`:

```
cookiecutter gh:VipulMascarenhas/sagemaker-template

```

The inputs for the template are described below:

| Input | Description                                                                                                                              |
| --- |------------------------------------------------------------------------------------------------------------------------------------------|
| PROJECT_NAME | Name of model project.                                                                                                                   |
| PY_VERSION | Which version of python to use, e.g. '3.11.3'.                                                                                           |
| DIR_MODEL_LOCAL | Local directory for model artifact storage, e.g. './artifacts'.                                                                          |
| DIR_TMP | Temporary files directory, e.g. '/tmp'.                                                                                                  |
| AWS_ACCOUNT_ID | 12-digit AWS account ID.                                                                                                                 |
| AWS_DEFAULT_REGION | AWS default region.                                                                                                                      |
| ECR_REPO | Name of AWS ECR repository, where containers are published.                                                                              |
| SAGEMAKER_ROLE | Name of the Sagemaker execution role to be assumed by Sagemaker, e.g. SageMakerRole                                                      |
| BUCKET_ARTIFACTS | Name of S3 bucket for model artifact storage. <br/>**NOTE**: prefix with 'sagemaker' for Sagemaker access, e.g. 'sagemaker-emails-data'. |

**NOTE**: do *NOT* enquote input values.

### Set-up project
Initialize project by executing `make init` from the command line in the project directory. The `init` target makes the included shell scripts executable and provisions relevant AWS infrastructure.

Export project-specific environment variables automatically with `direnv`, i.e. by invoking `direnv allow`.

### Template structure
To help you navigate in this template, here is an overview of the folder structure:

    ./
    ├── .github/    
    │   └── workflows/            # Workflows for automation, CI/CD.
    ├── modelpkg/                 # Python package defining model logic.
    |   |   construct.py          # Code for constructing and training the model etc.
    │   └── tests/                # Unit tests for model code.
    ├── aws/                      # Shell scripts for integrating the project with Sagemaker.
    ├── configs/                  # Configurations for Sagemaker endpoints, training jobs, etc.
    ├── images/                   # Docker images for model training and model endpoint.
    ├── server/                   # Configuration for a default NGINX web server for the model endpoint.*
    ├── .envrc                    # Project-specific environment variables.
    ├── Makefile                  # Command-line functions for project-specific tasks.
    ├── train.py                  # Script for training the model. Builds into training image.
    ├── app.py                    # Application code for the model endpoint. Builds into endpoint image.
    ├── requirements_modelpkg.txt # Python packages required by the model.
    └── requirements_dev.txt      # .. All other python packages needed in development mode.

We can modify this template depending on our specific use-cases.

### Command-line functions
All tasks related to interacting with the model project are implemented as command-line functions in `./Makefile` implying that functions are invoked with `make [target]`, e.g. `make build_training_image`.

If you want to build, train and deploy a model **on-the-fly** you can do it by invoking a sequence of `make` targets, i.e.:

1. `make init`
2. `make build_training_image`
3. `make push_training_image`
4. `make create_training_job`
5. `make build_endpoint_image`
6. `make push_endpoint_image`
7. `make create_endpoint`

`make` + <kbd>space</kbd> + <kbd>tab</kbd> + <kbd>tab</kbd> lists all available `make` targets.

### CI/CD workflows
This template ships with a number of automation (CI/CD) workflows implemented with Github Actions.

To enable CI/CD workflows, upload your project to Github and connect the Github repository with 
your AWS account by providing your AWS credentials as `Github` Secrets. Secrets should have names:

1. *AWS_ACCESS_KEY_ID*
2. *AWS_SECRET_ACCESS_KEY*

By default, every commit to `main` triggers a workflow `./github/workflows/deliver_images.yaml`, that runs unit tests and builds and pushes training and endpoint images. 

All workflows can be [run manually](https://docs.github.com/en/actions/managing-workflow-runs/manually-running-a-workflow).

## References

- [Github template](https://github.com/smaakage85/shapemaker)
- [Setting up SageMaker permissions](https://www.snowflake.com/blog/building-a-sagemaker-instance-from-scratch/)
- [How to train a model with Amazon SageMaker BYOC](https://www.sicara.fr/blog-technique/amazon-sagemaker-model-training)
- [How to create an endpoint with Amazon SageMaker BYOC](https://towardsdatascience.com/bring-your-own-container-with-amazon-sagemaker-37211d8412f4) 



