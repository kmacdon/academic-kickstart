---
title: "Building a new workflow with AWS and Docker"
summary: Setting up an easy to launch programming environment in AWS.
date: 2020-11-28
authors: ["admin"]
tags: ["python", "aws", "docker"]
output: html_document
---



## Introduction

While I was working on building a project involving the use of Keras earlier this year, I quickly realized my laptop was not powerful enough to build the models I was creating in a reasonable amount of time. I spent some time learning how to set up instances on AWS and using Docker enviroments to train my models with more powerful resources, and although it was fairly simple to do, I grew tired of having to set up my environment everytime. I'd have to launch an instance from the AWS website, ssh into it, download docker and the image I needed, launch the docker image, and then log in which ended up taking about 5 minutes everytime I wanted to work on my project. 5 minutes isn't a lot, but I knew there was a way I could automate it, so I decided to build that as another project. 
## Launching AWS Instance
Fortunately, Amazon has a python package called `boto3` that provides an API for ineracting with many AWS services, including S3 and EC2, so I decided to build a command line python tool around that (code can be found [here](https://github.com/kmacdon/aws-launcher)).

This part of the project actually ended up being pretty simple because I really only needed to specify 4 parameters when launching an instance:

1. Amazon Machine Image (AMI)
2. Instance Type
3. Security Group
4. Private Key

The AMI is just the default software that's installed when the instance is launched, and this provided an easy way for me to speed up my set up process. I used a standard Ubuntu image as my base, and then pre-installed Docker and downloaded the image (explained more in depth later) that I would be using. This helped shave off a minute or two from the process since the software I needed was already there, and any updates to the docker image would only require downloading updates instead of the whole thing.

The instance type I parametized so that I could specify the amount of resources I needed when launching, which I simplified to small, medium, or large based on the instances I had typically used. 

The security group and private key were things I had already created, with the security group only allowing access from my IP address and the private key obviously just being an ssh key.

I ended up just creating a JSON file for my base configuration like this:
``` json
{
    "ImageId":"ami-01b63af33be421653",
    "InstanceType": "t2.micro",
    "KeyName": "aws_key",
    "MaxCount": 1,
    "MinCount": 1,
    "Monitoring": {
        "Enabled": true
    },
    "SecurityGroups": [
        "my-ip-only"
    ],
    "IamInstanceProfile": {
        "Name": "aws_s3_access"
    }
}
```

Then whenever I run my command:
``` python
python start_aws.py --size <size>
```

The `InstanceType` variable will just get switched out with the corresponding size and all these parameters get passed to the `run_instance` function in the `boto3` package.

## Creating Docker Image
Setting up the Jupyter environment in Docker was actually really easy because there are already images available on Docker Hub that have environments with Jupyter and libraries like pandas, tensorflow, and scikit-learn pre-installed. 

The diffuclt part was getting them to work with my credentials for github and S3. Since I was going to have a fresh environment everytime I launched a new instance, I would have to download all my code and data each time. Now github obviously makes it easy to download code, and with the `boto3` package, it's really simple to download and upload data to/from S3, but I struggled with figuring out how to deal with authentication.

I could just copy my private key for github and secret keys for AWS into the Docker image because it's public and then others would have access to them which I obviously don't want. This part of the process was definitely the most frustrating and involved a lot of trial and error before I found a solution.

The AWS credentials were a lot simpler because I found out that the `boto3` package will check environment variables for your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. Fortunately, Docker makes it easy to pass environment variables at run time with the `-e` flag, so that issue was solved quickly.

The harder part was figuring out how to pass my private key for github, since I couldn't pass that as an environmental variable. After much googling, I found out that if I launch my image as part of a swarm instead of an individual container (even though I'm only running 1 version of it in the swarm) I can use a compose file to specify secrets.

The compose file is just a yaml file that defines the options for running the image, including environment variables and secrets which will get stored on the container in `/run/secrets/`. The compose file I use is defined below, so everytime I launch a container this way, my aws credentials which are defined in `aws_credentials.env` will be initialized in the container, and my private key stored in `aws_git` gets copied to the container in the folder listed above, which I then copy to my `.ssh` folder at launch in the startup script. 

```
services:
  jupyter-app:
    image: kmacdon/jupyter-env:latest
    ports:
      - "80:8888"
    environment:
      - JUPYTER_ENABLE_LAB=yes
    env_file: 
      - aws_credentials.env
    secrets:
      - git_ssh
secrets:
  git_ssh:
    file: ./aws_git
```

## Combining the Two
Now that I had these two building blocks, I ended up creating a simple SSH class using the `paramiko` and `scp` packages so that I could easily upload the credentials files needed by the image as well as running the commands to launch it without manually SSHing into the instance. Now, I can just run the one line command I specified above and then in the background the process will:

1. Launch an EC2 instance with my AMI and the specified size and security group
2. Upload my private key and AWS credentials to the server
3. Launch Docker and run the image, simultaneously updating it to the latest version and copying my credentials into the container
4. Copy my credentials into the appropriate locations in the container
5. Launch a Jupyter Lab session and open it through a browser on my computer 

All of this takes about 2 minutes and requires me to do nothing other than figure out how many resources I need and then wait. 

Overall, I love using this process now, and plan on creating another Docker image with RStudio instead so I'm not limited to Python. Even though I still like writing code locally and 95% of my projects run fine on my laptop, it's really enjoyable to explore the possibilities that Docker and cloud computing provide. As long as I have an internet connection, I'd be able to run all of my projects from anything.
