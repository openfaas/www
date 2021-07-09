---
title: Introducing OpenFaaS for AWS Lambda (faas-lambda)
description: Ed Wilde demonstrates how you can enjoy a developer-first experience on Kubernetes and AWS Lambda without the worry of vendor lock-in.
date: 2019-06-13
image: /images/faas-lambda/backdrop.jpg
categories:
  - lambda
  - aws
  - cloud
  - tutorial
  - kubernetes
author_staff_member: ed
dark_background: true
---

Learn how [AWS Lambda](https://aws.amazon.com/lambda/) and [OpenFaaS](https://github.com/openfaas/faas) can play well together with the new provider called faas-lambda. You'll learn how to deploy functions to [Kubernetes](https://kubernetes.io) and AWS with the same developer experience and workflow which is known and loved by the OpenFaaS community.

Introduction by: [Alex Ellis](https://www.alexellis.io)

> It was only a few months ago that I remember having lunch with Ed in a pub in the English country-side. We were throwing around ideas for a deeper integration between the AWS eco-system, and OpenFaaS. We wanted users to have more choice and to bring the OpenFaaS developer-experience to AWS customers. I'm excited to see how far Ed's got so far and I'm excited about what we've been able to spin out as new projects for the ecosystem.

Over to Ed.

## Outline

I'll start by exploring some of the use-cases I see for running OpenFaaS functions on the AWS Lambda platform. Then I'll take a look at some of the new components written to make this possible and how they work together. To wrap up, I'll ask the community for feedback and input on the future of faas-lambda.

## Why combine OpenFaaS and AWS Lambda?

You may be wondering what value there is in combining OpenFaaS with AWS Lambda. These are the top use-cases we came up with in the community:

- **A unified developer experience**

  Deploying AWS Lambda code normally requires a custom build and deployment pipeline involving many third-party tools and use of AWS CloudFormation. faas-lambda gives you the potential to eliminate this duplication and extra effort using one consistent pipeline to deploy to Kubernetes and AWS Lambda.

- **Journey to multi-cloud**

    Perhaps you started out your cloud journey on AWS and are looking to become vendor neutral and run on Kubernetes? With faas-lambda you can port your existing functions to OpenFaaS and still run them on AWS Lambda until you are ready to migrate them over to Kubernetes

- **Proprietary triggers & integrations**

  There are some integration points in the AWS architecture that mandate the use of Lambdas. 
However, there is no reason you can't create, manage and run these function using OpenFaaS with it's enhanced developer experience.


- **Free-tier, specialized hardware and extended limits**

  AWS comes with a free-tier for invocations at a certain duration/memory limit, which you could use to save on costs. When you hit the limits of AWS Lambda, you can break out into your own Kubernetes cluster and provide additional disk space, GPU support, iops, CPU and memory.

Here's a clipping from my presentation at the [Serverless Practitioners Summit](https://www.openfaas.com/blog/meet-us-at-barcelona/) at KubeCon, Barcelona. It shows some of the differences between OpenFaaS on Kubernetes and AWS Lambda.

![](/images/faas-lambda/faas-lambda-compare.png)

*Comparison AWS Lambda vs OpenFaaS with Kubernetes*

The great news is that you no-longer have to pick between one, or the other. You can get the best of both worlds with a single, unified control-plane.

## Conceptual architecture

OpenFaaS can be extended through the use of interfaces. The most common way to extend OpenFaaS is to create a provider.

The provider implements a set REST API:

* CRUD for functions
* Scale function
* Query function readiness
* Invoke function

You can use the [faas-provider SDK](https://github.com/openfaas/faas-provider) to build your own provider for any back-end, even an [in-memory datastore](https://github.com/openfaas-incubator/faas-memory) in a very short period of time.

> To find out more about faas-provider read [Power of Interfaces in OpenFaaS](https://blog.alexellis.io/the-power-of-interfaces-openfaas/) by Alex.


This interface is what inspired me to create faas-lambda.

![faas-lambda conceptual architecture](/images/faas-lambda/faas-lambda-provider.png)

*Conceptual architecture*

The provider I wrote understands how to translate requests from the gateway into appropriate 
responses in the AWS Lambda world.

In my reference architecture the faas-lambda provider runs inside a Kubernetes cluster
as a Pod along with the existing OpenFaaS core serviecs. I then used the [AWS SDK for Go](https://aws.amazon.com/sdk-for-go/) to communicate with the AWS Lambda API. faas-lambda is currently configured to target a single AWS account per installation, so I store my credentials in a Kubernetes *Secret* object. When running with [Amazon's Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks) it would be possible to remove the secrets and use IAM roles instead.

> See also: [kiam](https://github.com/uswitch/kiam) by uswitch, which can be used for IAM on EKS.

### Deploy a function

OpenFaaS functions are built into immutable Docker or OCI-format images, which are then stored in a registry such as the Amazon Elastic Container Registry (ECR). The first challenge I had was how to transform layers in my Docker image into an AWS Lambda package.

This is how I decided to approach the problem. There is also potential to use Lambda layers in the future, but for now the provider creates a single layer or archive per function.

![Repackaging a Docker image to a Lambda](/images/faas-lambda/faas-lambda-deploy.png)

*Repackaging a Docker image to a Lambda*

1. Using the [Skopeo](https://github.com/containers/skopeo) Go SDK we can download the container image of the function at deployment time
2. Next, we extract each of the container layers using the image manifest information we've obtained from Skopeo.
3. Because the AWS Lambda function has a different handler signature, we need to create a small shim. This shim will allow the AWS Lambda service to call into our OpenFaaS function during invocation.
4. Finally, we need to make sure that all of the libraries the function depends on, are in the correct location the AWS Lambda runtime will expect them to be in. Once that's done, we zip up the shim, original function and libraries to create our AWS Lambda package.

> Note: Step 3 could be avoided by simply creating an OpenFaaS template with a signature that matches the AWS signature. In this exercise, we wanted to specifically offer interoperability with existing OpenFaaS functions.

* What about native dependencies?

Native dependencies can be tricky, but by building a new OpenFaaS template with the base layer from the [Docker Lambda](https://github.com/lambci/docker-lambda) project, we could build native dependencies which would be compatible with the AMI used for the Lambda runtime.

### Invoke a function

This was an easier problem for me to solve. Using the AWS for Go SDK, we simply call the lambda function, passing in the request body we received from the client. We wait for the function to complete and proxy back the response. 

You may have noticed that we've not needed to use the AWS API gateway. This can saving you money, reduce complexity and allow your function to execute for up to 15 minutes, which is the current AWS Lambda limit. If you use API gateway the current execution time is limited to 30 seconds.

The other thing this enables us to do is to return any content-type we like, including HTML. AWS Lambda at time of writing is limited to turning `application/json`.

![Invoking an OpenFaaS function on AWS Lambda](/images/faas-lambda/faas-lambda-invoke.png)

## Best of both worlds: AWS Lambda & Kubernetes

The new component allows us to run functions on AWS Lambda, or on Kubernetes. I wanted to find a way to have a single deployment of the OpenFaaS gateway which could target either platform using a scheduling hint or annotation such as "this function needs to go to AWS".

I had a quick design session with Alex and we came up with the idea for `faas-federation`.

> See the code: [faas-federation](https://github.com/openfaas-incubator/faas-federation)

![enable multiple execution platform with faas-federator](/images/faas-lambda/faas-federation-provider.png) 

The `faas-federation` implements the same provider API that I used for faas-lambda and that is used by the core project for faas-netes (OpenFaaS on Kubernetes). So whenever an invocation is made or a deployment requested, faas-federation is in the hot-path and can act like a router, picking one path or the other.

> We're also excited about using `faas-federation` for implementing an East/West split and for bridging edge-locations to a larger control plane. Imagine that your Raspberry Pi clusters can tap into extra compute in a central location.

Whilst flying over to Chicago with Alex to present and run a workshop on Serverless Kubernetes, I wrote 90% of the code for a new provider called [faas-memory](https://github.com/openfaas-incubator/faas-memory). It allows any OpenFaaS platform developer or contributor to test against or develop against the OpenFaaS REST API whilst offline and without a real Kubernetes cluster.

This is how simple the code was for the "create" handler for the in-memory provider:

```go
package handlers

import (
	"encoding/json"
	"io/ioutil"
	"net/http"

	"github.com/openfaas/faas/gateway/requests"
	log "github.com/sirupsen/logrus"
)

var functions = map[string]*requests.Function{}

func MakeDeployHandler() http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {

		defer r.Body.Close()

		body, _ := ioutil.ReadAll(r.Body)

		request := requests.CreateFunctionRequest{}
		if err := json.Unmarshal(body, &request); err != nil {
			log.Errorln("error during unmarshal of create function request. ", err)
			w.WriteHeader(http.StatusBadRequest)
			return
		}

		functions[request.Service] = createToRequest(request)
		w.WriteHeader(http.StatusOK)
	}
}
```
*[deploy.go](https://github.com/openfaas-incubator/faas-memory/blob/master/handlers/deploy.go)*

Contributions to faas-memory and faas-provider are both welcome and the projects are Open Source under the MIT license.

## Demo

Here's a demo of it all put together. I'll use OpenFaaS to deploy, manage and invoke functions on both Kubernetes and AWS Lambda:

{% include youtube.html id="Dv9BEQWnkRQ" %}

Language support for Go, Node and Python is available and we will be adding more language support through the use of the [Lambda Custom Runtime](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html). You'll have a chance to register interest below.

## What's next

Contributions to faas-memory, faas-federation and faas-provider are both welcome and the projects are Open Source under the MIT license.

Here are some of the projects we've talked about

* [faas-memory](https://github.com/openfaas-incubator/faas-memory) - in-memory provider (MIT)
* [faas-federation](https://github.com/openfaas-incubator/faas-federation) - provider router (MIT)
* [faas-provider](https://github.com/openfaas/faas-provider) - provider SDK (MIT)
* [faas-netes](https://github.com/openfaas/faas-netes) - for Kubernetes (MIT)
* faas-lambda - early preview

To register your interest and give feedback sign-up below:

* [faas-lambda early preview](https://docs.google.com/forms/d/e/1FAIpQLSe9AfH-uCuK_tUeeRQWLe7_Ti-lbMF5iynwbdfszmB9NEttow/viewform)

### Get connected

Are you using OpenFaaS in production? [Join the end-user community](https://github.com/openfaas/faas/issues/776) or [sponsor the project](https://www.openfaas.com/support/) to give back. Financial backing is required to ensure its on-going.

If you have comments, questions or suggestions or would like to join the community, then please [join us on OpenFaaS Slack](https://docs.openfaas.com/community/).

You can follow [@openfaas on Twitter](https://twitter.com/openfaas/)

Editor: [Alex Ellis](https://www.alexellis.io)
