---
layout: post
title:  "First Impressions of CloudFormation"
date:   2017-09-06 10:17:00 -0400
categories: aws cloudformation
---
I've been using Cloudformation for some projects after a long time working
with [Terraform](https://terraform.io).  The hope is that switching to an
all AWS ecosystem would yield a better set of tools than Terraform, which is
multi-provider.

I like that Cloudformation is a module-first approach that lets you re-use
templates for lots of similar resources.  I also like that Cloudformation
runs in AWS, so unlike Terraform you don't have inconsistencies between
versions or environment setup.  All that is abstracted away.

### Error reporting
One of the gotchas about Cloudformation's stack approach is that errors aren't
bubbled up in full detail to the parent stack.  It took me a long time to
realize that there was more detail than what was on the parent stack.

Clicking out, finding the offending child stack, and clicking into that is a lot
less intuitive than the way any and all Terraform errors are reported at the
end of a failed run.

### The Documentation is Tough
Maybe it's just me, but I really glean a lot from examples.  Show me the way
it's supposed to work, and then I don't have to parse whether a reference to
another component is supposed to be an attribute or an ARN or whatever.

Cloudformation docs are nearly devoid of useful examples, whereas Terraform
doc pages lead off with one or more fairly fully complete examples.

### Rollbacks are Nice, but Slow for Developing
When rolling out changes to a mature stack rollbacks are critical.  If
something goes wrong, roll back to the previous good state so the issue can
be resolved.  Delightful!

But when you're spinning up a new stack, man it can take a long time.  Make a
change, start the Cloudformation stack changes, wait.  Catch an error.  Wait
for rollbacks to complete.  Especially on resources that take a long time to
provision, like Route53 and Elasticache, or Cloudfront, then these cycles can
take forty-five minutes each, which leaves you constantly switching contexts
and at the end of the day without much progress to show.
