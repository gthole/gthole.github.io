---
layout: post
title:  "Some Limitations of AWS Lambda"
date:   2016-08-10 16:15:42 -0400
categories: aws lambda
---
**Update 5/18: Many of these gripes aren't really applicable anymore.**

I've been working with AWS Lambda for a couple months now, porting Enernoc's
data ingestion architecture to a serverless-style stack.  I love the simplicity
and flexibility of Lambdas and automating the deployment.  Management through
Terraform makes the whole experience push-button once you have the initial
configurations written.

There are some real limits to what Lambda works best for, that I've run up
against in my zeal.

### Network Performance can be Terrible
We have a simple ETL script to mock live data streams for demonstration
purposes - it pulses out adjusted readings every 5 minutes from a batch of
CSV files that contains the base readings.

There's a pre-load step to this script that retrieves the CSV from our
enterprise Github server (they're in VCS so project managers can edit the
base readings to suit their demonstrations).  Pulling down a single CSV on my
work laptop would take a few seconds, but running that same retrieval step in
Lambda took over two minutes, despite being "closer" within the network to
the Github server.

As result, we couldn't reliably do the pre-load step in Lambda anymore, but had
to store the compiled base readings in the built zip file as Avro blobs.  This
will ultimately be more cost-efficient anyway so that we're not paying AWS to
do the underlying transformation, but it's disheartening to have to find
work-arounds for poor network performance.

### ETL functions and Timeouts
The 5 minute cap on Lambda execution time eliminates a whole lot of excellent
potential uses.  Something I'd really like to see would be a plugin architecture
to launch [Scrapy]() jobs to crawl websites via Lambda.  But if these crawls are
particularly extensive or paced to give time to the site it's crawling, it will
be easy to run up against that 5 minute timeout mark.

An interesting work-around might be to have recursive lambda calls - performing
a depth-first crawl where each entry into the next level down (or up ... but "the
enemy's gate is down") triggers a new Lambda invocation with the arguments and
context to perform that section.  Make them asynchronous and you've got a
clean parallel execution stack.

One might even work in timeouts and back-off to reduce remote server load by
triggering child lambda calls with SNS messages.

### Polling via Cloudwatch Events
Want to run your Lambda on a cron?  You're doing it with Cloudwatch Events.
But you can't trigger Lambdas at a rate higher than once per minute.  As such,
using Cloudwatch Events to poll an SQS queue (for example) will introduce a
minute-lag in your pipeline.

Maybe you could hook up an SNS topic to trigger whenever the SQS queue depth
hits a certain (relatively low) limit, but it'd be more ideal for Lambda to
have a polling event trigger for SQS the way it does for Kinesis.

### Environment Configurations
Lambda supports versions and aliases to help promote code through environments
from staging to production.  But any built code that's being promoted will have
configuration differences between environments, and yet there's no feature
support for setting environment variables on Lambda functions.

Usually we end up bundling the configuration in with the built zip file. An
alternative would be to make a network call to a service like Zookeeper, Consul,
or Dynamo to retrieve configs at start-up time, but that's awkward and can be
prohibitive on Lambdas that need fast container initialization (like backing
API Gateway).

Setting environment variables would move the scope of configuration into the
realm of provisioning and deployment, which is where it belongs, and would
be nice and fast local retrieval for quick-start containers.  I hope the AWS
Lambda team gets this feature out soon!
