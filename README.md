# AWS EC2 Client Package

**aws.ec2** is a simple client package for the Amazon Web Services (AWS) [Elastic Cloud Compute (EC2)](http://aws.amazon.com/ec2/) REST API, which can be used to monitor use of AWS web services.

To use the package, you will need an AWS account and to enter your credentials into R. Your keypair can be generated on the [IAM Management Console](https://aws.amazon.com/) under the heading *Access Keys*. Note that you only have access to your secret key once. After it is generated, you need to save it in a secure location. New keypairs can be generated at any time if yours has been lost, stolen, or forgotten. The [**aws.iam** package](https://github.com/cloudyr/aws.iam) profiles tools for working with IAM, including creating roles, users, groups, and credentials programmatically; it is not needed to *use* IAM credentials.

To use the package, you will need an AWS account and to enter your credentials into R. Your keypair can be generated on the [IAM Management Console](https://aws.amazon.com/) under the heading *Access Keys*. Note that you only have access to your secret key once. After it is generated, you need to save it in a secure location. New keypairs can be generated at any time if yours has been lost, stolen, or forgotten. The [**aws.iam** package](https://github.com/cloudyr/aws.iam) profiles tools for working with IAM, including creating roles, users, groups, and credentials programmatically; it is not needed to *use* IAM credentials.

A detailed description of how credentials can be specified is provided at: https://github.com/cloudyr/aws.signature/. The easiest way is to simply set environment variables on the command line prior to starting R or via an `Renviron.site` or `.Renviron` file, which are used to set environment variables in R during startup (see `? Startup`). They can be also set within R:

```R
Sys.setenv("AWS_ACCESS_KEY_ID" = "mykey",
           "AWS_SECRET_ACCESS_KEY" = "mysecretkey",
           "AWS_DEFAULT_REGION" = "us-east-1",
           "AWS_SESSION_TOKEN" = "mytoken")
```


## Code Examples

The basic idea of the package is to be able to launch and control EC2 instances. You can read [this blog post from AWS](https://blogs.aws.amazon.com/bigdata/post/Tx3IJSB6BMHWZE5/Running-R-on-AWS) about how to run R on EC2.

A really simple example is to launch an instance that comes preloaded with an RStudio Server Amazon Machine Image ([AMI](http://www.louisaslett.com/RStudio_AMI/)):

```R
# Describe the AMI (from: http://www.louisaslett.com/RStudio_AMI/)
image <- "ami-3b0c205e" # us-east-1
#image <- "ami-93805fea" # eu-west-1
describe_images(image)

# Check your VPC and Security Group settings
## subnet
subnets <- describe_subnets()
## security group
my_sg <- create_sgroup("r-ec2-sg", "Allow my IP", vpc = subnets[[1L]])
## use existing ip address or create a new one
ips <- describe_ips()
if (!length(ips)) {
    ips[[1L]] <- allocate_ip("vpc")
}

# create an SSH keypair
my_keypair <- create_keypair("r-ec2-example")
pem_file <- tempfile(fileext = ".pem")
cat(my_keypair$keyMaterial, file = pem_file)

# Launch the instance using appropriate settings
i <- run_instances(image = image, 
                   type = "t2.micro", # <- you might want to change this
                   subnet = subnets[[1L]], 
                   sgroup = my_sg,
                   keypair = my_keypair)
Sys.sleep(5L) # wait for instance to boot

# associate IP address with instance
instance_ip <- get_instance_public_ip(i)
if (is.na(instance_ip)) {
    instance_ip <- associate_ip(i, ips[[1L]])$publicIp
}
# authorize access from this IP
try(authorize_ingress(my_sg))
try(authorize_egress(my_sg))
```


With the instance up and running, you can now access it a number of ways. Given it's an RStudio Server instance, you can just navigate to the public IP address via your browser:

```R
# open RStudio server in browser
browseURL(paste0("http://", instance_ip))
```

Or you can login via `ssh` or `putty` on the command line. Thanks to the [**ssh**](https://cran.r-project.org/package=ssh) package, you can also create an SSH connection directly in R:

```R
# log in to instance
library("ssh")
session <- ssh::ssh_connect(paste0("ubuntu@", instance_ip), keyfile = pem_file, passwd = "rstudio")

# write a quick little R script to execute
cat("'hello world!'\nsprintf('2+2 is %d', 2+2)\n", file = "helloworld.R")
# upload it to instance
invisible(ssh::scp_upload(session, "helloworld.R"))

# execute script on instance
x <- ssh::ssh_exec_wait(session, "Rscript helloworld.R")

## disconnect from instance
ssh_disconnect(session)
```

Another option is to again use **ssh** to setup an interactive session that will be usable with [**remoter**](https://cran.r-project.org/package=remoter). This requires running

```R
remoter::server()
```

on the instance and then logging into it locally with `remoter::client()`. This is left as an exercise for advanced users.

Regardless of how you access the instance, make sure to turn it off and clean up after yourself:

```R
## stop and terminate the instance
stop_instances(i[[1L]])
terminate_instances(i[[1L]])

## revoke access from this IP to security group
try(revoke_ingress(my_sg))
try(revoke_egress(my_sg))

## delete keypair
delete_keypair(my_keypair)

## release IP address
release_ip(ips[[1L]])
```


## Installation

[![CRAN](https://www.r-pkg.org/badges/version/aws.ec2)](https://cran.r-project.org/package=aws.ec2)
![Downloads](https://cranlogs.r-pkg.org/badges/aws.ec2)
[![Travis Build Status](https://travis-ci.org/cloudyr/aws.ec2.png?branch=master)](https://travis-ci.org/cloudyr/aws.ec2)
[![codecov.io](https://codecov.io/github/cloudyr/aws.ec2/coverage.svg?branch=master)](https://codecov.io/github/cloudyr/aws.ec2?branch=master)

This package is not yet on CRAN. To install the latest development version you can install from the cloudyr drat repository:

```R
# latest stable version
install.packages("aws.ec2", repos = c(getOption("repos"), "http://cloudyr.github.io/drat"))
```

Or, to pull a potentially unstable version directly from GitHub:

```R
if(!require("remotes")){
    install.packages("remotes")
}
remotes::install_github("cloudyr/aws.ec2")
```


---
[![cloudyr project logo](http://i.imgur.com/JHS98Y7.png)](https://github.com/cloudyr)
