---
layout: post
title:  "Ditch Jenkins for Serverless DevOps"
date:   2018-05-30 17:00:00 -0400
categories: aws codebuild jenkins
---

At EnerNOC we have a lot of AWS accounts.  We have different accounts for different
teams, so that nobody accidentally steps on each other.  Then, each team has
both a prod and a dev account.  It's not the most ideal approach, but does
give us a lot of sandboxes.

Each account has at least one Jenkins instance for dev automation.  You know
the whole "pet versus cattle" thing, right?  A pet instance is a server
where you love it and take care of it. But cattle instances are disposable.
Something goes wrong?  Just nuke it and move on with life.

Jenkins instances are pet instances.  You get enough of them together, and
suddenly you have a whole flock of pets.  That's a lot of care and management
of your devops processes.

### Enter CodeBuild and Terraform

AWS Codebuild lets you perform long-running builds triggered by Github webhooks.

It's built on top of ECS, so you can start with one of the AWS curated images
or bring your own Docker image as the base image from which to build.  You pay
half a cent per minute, instead of having always on capacity.

Github triggers your builds via Webhooks - someone opens a pull request?
Automatically run tests on it!  Merge the PR?  Automatically build/deploy it.

### Build Commands as Code
Codebuild operates off of [buildspec.yml](
https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)
files in your project.  The core of this file is the "phases" block where you
supply commands to perform your build.

For a docker-based project this might mean building the Dockerfile and pushing
to ECR.

For example:

```
version: 0.2

phases:
  pre_build:
    # Log ECR so we can pull and push images respectively
    commands:
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    # Build and deploy
    commands:
      - export SHA=`git rev-parse HEAD`
      - export REPO=1234567890.dkr.ecr.us-east-1.amazonaws.com
      - docker build -t $REPO/my-project:$SHA .
      - docker push $REPO/my-project:$SHA
```

There's a lot of beauty in this - you're checking your build process into VCS
instead of having it in a Jenkins configuration that might go away if the
Jenkins instance fails.  Of course there's the `Jenkinsfile`, but honestly
I'd much rather write a YAML file.

### DevOps Configurations as Code

Even better, since this is all set up with Github and AWS, Terraform can manage
the configurations for Codebuild and the Github webhooks. Drop a `buildspec.yml`
file into your project, then create a Terraform manifest file to create and
manage the Codebuild project and webhooks from Github.  The following example
is for a Github Enterprise setup, with a docker based workflow:


```
##############################
# Auto Deploy Develop Branch #
##############################

resource "aws_codebuild_project" "my_project_deploy" {
    name = "my-project-deploy"
    build_timeout = "60"
    service_role = "${var.codebuild_execution_arn}"

    artifacts {
        type = "NO_ARTIFACTS"
    }

    cache {
        type = "NO_CACHE"
    }

    environment {
        compute_type = "BUILD_GENERAL1_SMALL"
        image = "aws/codebuild/docker:17.09.0"
        privileged_mode = true
        type = "LINUX_CONTAINER"

        environment_variable {
            "name" = "ENV"
            "value" = "stage"
        }
    }

    source {
        type = "GITHUB_ENTERPRISE"
        location = "https://github.my-company.com/My-Org/my-project.git"
        buildspec = "./buildspec.yml"
        auth {
            type = "OAUTH"
            resource = "${var.github_token}"
        }
    }

    vpc_config = ["${var.vpc_config}"]
}

resource "aws_codebuild_webhook" "my_project_deploy" {
    project_name = "${aws_codebuild_project.my_project_deploy.name}"
    branch_filter = "develop"
}

resource "github_repository_webhook" "my_project_deploy" {
    repository = "my-project"
    name = "web"

    configuration {
        url = "${aws_codebuild_webhook.my_project_deploy.payload_url}"
        secret = "${aws_codebuild_webhook.my_project_deploy.secret}"
        content_type = "json"
        insecure_ssl = false
    }

    active = true
    events = ["push"]
}
```

Of course you still have to set up the Terraform state and everything, but then
even your CI integrations are part of your VCS.

But really, serverless DevOps is pretty great!  Instead of paying for an
always-on Jenkins instance (that you have to love and care for all the time),
provision a CodeBuild project, and pay for it by usage, with everything tracked
in git.

Is it better than CircleCI?  Maybe, maybe not.  But for teams who work in
dedicated AWS shops, it sure is better than managing a bunch of Jenkins
instances.
