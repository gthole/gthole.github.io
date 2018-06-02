---
layout: post
title:  "Terraform and API Gateway"
date:   2016-07-15 12:12:31 -0400
categories: aws api gateway lambda terraform
---
AWS API Gateway is pretty neat.  But manually managing the configurations is
a nightmare - you make changes in your development stack to fix a bug or support
something new, do that a bunch of times for a sprint, and when it's time to
push out the release candidate, each one of those changes has to be tracked
and updated in the console.

Even for an API with only a few endpoints and no authorizer, my team was often
caught out by forgetting some change, finding that our deployment configs
were different between stacks, or stepping on each other's changes in the
console.

### Step 1: Move everything into Terraform
Treating deployment configurations as code is what Terraform does best, so we
looked up API Gateway support in the AWS provider, and hooray it was just added!

API Gateway effectively comes down to three primary configuration points:

- APIs: the top level resource for an API
- Deployments: where a batch of configurations under an API is actually made
  available to the internet
- Resources: the individual endpoints

Each resource itself is a block of configurations:

- Method: Defining the HTTP verb (GET, PUT, etc.), headers and parameters to
  expect.
- Integration: How to parse and pre-process the body, headers, and parameters
  into a request to pass on to the back end, and what back end to connect to
  (e.g. proxy to an internal service, call a Lambda, send a fixture)
- Integration Response: Convert the return value from the back end into an
  appropriate HTTP response, with status codes, headers, and response body.
- Method Response: Flag statuses and headers to return

The Terraform implementation treats each bullet point in the above two lists
as a separate resource block to be managed by Terraform and synced to the
AWS API.  For example:

```hcf
resource "aws_api_gateway_rest_api" "api" {
    name = "My fancy API"
    description = "It does lovely things"
}

resource "aws_api_gateway_deployment" "v1" {
    stage_name = "v1"
    rest_api_id = "${aws_api_gateway_rest_api.api.id}"
}
```

Now all those configurations are part of the Github repo for the project, and
tracked with changes that come in from developers.  Plus with Terraform we
could set Jenkins to watch the repository and automatically deploy to AWS
whenever we merged a pull request to `develop`.

### But then, after the Honeymoon ...
But as soon as we really started rolling with this, it started to become clear
that Terraform was buggy and missing features.  The highlights:

##### **Configuration Overload**
Each configuration above being a separate resource was pretty ungainly.  An
endpoint with four methods (say `GET`, `POST`, `PUT`, `DELETE`) and three
status code responses each (say `200`, `400`, `500`) results in 32 separate
resources in Terraform.  That's a lot of repeated code and a lot of API calls
to AWS that Terraform has to make to keep them all in sync (which means slow
deployments and big state files.)

##### **No Re-Deploy Once Deployed**
Once created, a Deployment resource is not re-deployed after configuration
changes get pushed up.  So after we run a deploy, someone still had to
remember to go into the console and press the "deploy" button.

##### **Life-cycle Dependency Woes**
Changes to an Integration resource would destroy the Integration Response
and Method Response resources underneath it, but the Terraform lifecycle
(sync first, create/update/destroy as needed) wouldn't notice the change.  As
result we had to `terraform apply` twice to make sure those resources got
re-created under the altered Integration resource.

##### **New Functionality, Limited Completeness**
At the time, query parameters and headers weren't supported, which was a
pretty big bummer to not be able to send Authorization headers or other
parameters to the API.  But some timely pull requests to Terraform helped
fix that.  Even so, the Terraform support was quite new and green.

### **But What About Swagger?**
API Gateway supports both export and import of Swagger docs to retrieve and
define the configuration of an API Gateway api.  Uploading a Swagger doc is
much more feature complete than Terraform resources, and doesn't suffer from
life-cycle dependencies the way Terraform does.  Since we want to use the
Swagger doc as documentation of the API after deployment, it'd be nice to use
the upload tool and let the documentation define the API with a contract-first
approach to deployment.

Our Lambdas and other resources work quite nicely in Terraform, so what's a
developer to do?  Script it out, of course!

I put the `swagger.yml` in the Github repo, create a `null_resource` Terraform
resource that watches it for changes, and runs a little NodeJS script to inject
Terraform variables into the Swagger doc and upload it to API Gateway whenever
there are changes.

This generously fixes all the issues mentioned above, and lets our documentation
and configuration manifests be the same thing.  As an aside too, with a little
`tv4` magic too, the API Gateway model resources (which are generally useless
in API Gateway) can perform validation of the payload requests to ensure the
payloads match the API Gateway model JSON schema.

### Where to Go From Here
Running the deployment as a script from Terraform works well, but I'd like to
see it as part of the core Terraform resource.

Something like:
```
resource "template_file" "swagger" {
    template = "${yaml_to_json(file("./swagger.yml"))}"
    vars {
        backend_lambda = "${aws_lambda_function.controller.arn}"
    }
}

resource "aws_api_gateway_rest_api" "api" {
    name = "${var.rest_api_title}"
    swagger = "${template_file.swagger.rendered}"
}

resource "aws_api_gateway_deployment" "v1" {
    stage_name = "v1"
    rest_api_id = "${aws_api_gateway_rest_api.api.id}"
    re_deploy = true
}
```
**NOTE**: If you're just scanning this article for code, the above is speculative
and will not work.  Take a look at this [gist]() instead.

I'd also like to see the API Gateway Lambda upload support `$ref` references
to common status code responses and more recent Swagger V2 support.  Also worth
noting that the response messages that API Gateway's `putRestApi` endpoint
returns can be confusing - I kept getting `415 Too Many Requests` responses
when I had a malformed swagger due to a typo.  Fixing the typo fixed the error
response, which had actually nothing to do with rate limiting.
