---
title: "Kubernetes Webhooks made easy with OpenFaaS"
description: "In this post you'll learn how to write Kubernetes Admission webhooks using OpenFaaS functions"
date: 2020-10-27
image: /images/2020-10-27-k8s-validatingwebhook-openfaas/puzzle.jpg
categories:
  - arkade
  - kubectl
  - faas-cli
  - admissionwebhooks
  - validatingadmissionwebhooks
  - k8s extensibility
author_staff_member: batuhan
dark_background: true
---

In this post you'll learn how to write Kubernetes Admission webhooks using OpenFaaS functions

<p align="center">
<img height="128" src="/images/openfaas/kubernetes.png">
</p>

## Introduction to Kubernetes Admission webhooks
Admission webhooks are HTTP callbacks that receive admission requests and do something with them. You can define two types of admission webhooks, validating admission webhook and mutating admission webhook. Mutating admission webhooks are invoked first, and can modify objects sent to the API server to enforce custom defaults. After all object modifications are complete, and after the incoming object is validate by the API server, validating admission webhooks are invoked and can reject requests to enforce custom policies.

Using OpenFaaS in this design, we can focus on our core logic more than designing the microservice itself and simply create application without being worry about how to build and deploy. 

## The Scenario
Let's assume, in our company, we have some requirements that we must meet while deploying applications onto the Kubernetes cluster. We need to set some required labels to our Kubernetes manifest. Unless we specify the required labels our request will reject.

So, in order to apply those requirements to the Kubernetes cluster to ensure the best practices, we can use Kubernetes [ValidatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook) and [OpenFaaS](https://www.openfaas.com) together. Since ValidatingAdmissionWebhooks intercepts requests to the apiserver, OpenFaaS functions includes a little code to check required labels and determines the request either allowed or not.

Webhook Admission Server is just plain http server that adhere to Kubernetes API. For each Pod create request to the apiserver(I said Pod because we specify which kind of resources that we consider while registering our webhook to the apiserver using [ValidatingWebhookConfiguration](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#validatingwebhookconfiguration-v1-admissionregistration-k8s-io) resource) the ValidatingAdmissionWebhook sends an admissionReview([API](https://github.com/kubernetes/kubernetes/blob/release-1.19/pkg/apis/admission/types.go) for reference) to the relevant webhook admission server. The webhook admission server gathers information like object, oldobject, and userInfo from admissionReview's AdmissionRequest and sends AdmissionRequest to the serverless function through the OpenFaaS Gateway. The function checks the required labels exist on Pod and determines the request either valid or not and then sends back the AdmissionResponse whose Allowed and Result fields are filled with the admission decision to the webhook admission server then the webhook admission servers sends back a admissionReview to the apiserver.

* Kubernetes API -> Webhook (w/TLS) -> OpenFaaS Gateway (w/HTTP) -> OpenFaaS Function

![Workflow](/images/2020-10-27-k8s-validatingwebhook-openfaas/admission-controller-phases.png)
> Credit: [https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/)

Supporting TLS for external webhook server is also required because admission is a high security operation. As part of the process, we need to create a TLS certificate signed by the Kubernetes CA to secure the communication between the webhook server and apiserver.

### Prerequisites
##### Arkade

* [arkade](https://get-arkade.dev) is The OpenFaaS community built tool for Kubernetes developers, with arkade you can easily install all necessary cli tools to your host and deploy apps to the cluster.

```sh
$ curl -sLS https://dl.get-arkade.dev | sudo sh
```

##### KinD (Kubernetes in Docker)

*  Kubernetes is our recommendation for teams running at scale, but in this demo we will be using [KinD](https://kind.sigs.k8s.io/docs/user/quick-start/) for the sake of simplicity.

```sh
$ arkade get kind
```

##### kubectl

* You can control your cluster using [kubectl](https://github.com/kubernetes/kubectl) CLI.

```sh
$ arkade get kubectl
```

##### faas-cli

* [faas-cli](https://github.com/openfaas/faas-cli) is an official CLI for OpenFaaS , with "faas-cli" you can build and deploy functions easily.

```sh
$ arkade get faas-cli
```

### Setup

### 1. Setup a Kubernetes Cluster with KinD

You can start a Kubernetes cluster with KinD if you don't have one already

```bash
$ arkade get kind
$ kind create cluster
```

### 2. Deploy OpenFaaS to our local Kubernetes Cluster with arkade:

* Install a OpenFaaS

```sh
$ arkade install openfaas
```

Read the output from the installation and run the commands given to you.

You can access them again at any time with `arkade info openfaas`

### 3. Clone the project

* Clone the sample from GitHub

```sh
$ git clone https://github.com/developer-guy/admission-webhook-example-with-openfaas
$ cd admission-webhook-example-with-openfaas
```

* Let's explore the structure of the project.

```
deployment/ --> includes necessary manifests and scripts for the deployment of the project
functions/ --> includes templates and the requiredlabel function itself
Dockerfile --> includes instructions to build an image of the project
build --> automated way to build and push an image of the project
```

### 4. Deploy ValidatingAdmissionWebhook

* We will generate the TLS certificates required for the ValidatingAdmissionWebhook using the following: [Kubernetes TLS Certificates Management](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)

```sh
$ cd deployment
$ sh webhook-create-signed-cert.sh
```

* Get the Certificate Authority (CA) from the local cluster

```sh
$ export CA_BUNDLE=$(kubectl config view --minify --flatten -o json | jq -r '.clusters[] | select(.name == "'$(kubectl config current-context)'") | .cluster."certificate-authority-data"')
$ sed -e "s|\${CA_BUNDLE}|${CA_BUNDLE}|g" validatingwebhook.yaml | kubectl apply -f -
$ cd ..
```

* Build the project

```sh
$ export DOCKER_USER="docker-hub-username"
$ ./build
```

Now edit `deployment.yaml` and set 'DOCKER_USER' to the above.

* Deploy it project

```sh
$ cd deployment
$ kubectl apply -f rbac.yaml,service.yaml,deployment.yaml
```

* Label the default namespace to enable the webhook

```sh
$ kubectl label namespaces default admission-webhook-example=enabled
```

### 5. Build and Deploy OpenFaaS Function (Optional)

* Pull the [golang-middleware](https://github.com/openfaas-incubator/golang-http-template) template from [OpenFaaS Official Template Store](https://github.com/openfaas/store)

```sh
$ faas-cli template store list # check available templates in store
$ faas-cli template store describe golang-middleware # describe the specific template
$ faas-cli template store pull golang-middleware
```

* Create the function

```sh
$ export OPENFAAS_PREFIX=$DOCKER_USER
$ faas-cli new requiredlabel --lang go-middleware
$ cd requiredlabel
$ go mod init requiredlabel
$ # fill the handler.go with the corresponding code
$ go get
```

* Deploy the function

```sh
$ cd functions
$ faas-cli up -f requiredlabel.yml --build-arg GO111MODULE=on # (build-push-deploy) make sure you are using your docker hub username. i.e: devopps
```

* Verify the functions that are working in `openfaas-fn` namespace

```sh
$ kubectl get pods --namespace openfaas-fn
```

### 6. Test the whole workflow

* The purpose of this PoC is that to validate that pods has required `labels`. Which means you must have that labels:

```yaml
app.kubernetes.io/name: sleep
app.kubernetes.io/instance: sleep
app.kubernetes.io/version: "0.1"
app.kubernetes.io/component: dummy
app.kubernetes.io/part-of: admission-webhook-example
app.kubernetes.io/managed-by: kubernetes
```

* Any Pod who have above labels is valid for us.

```sh
`deployment/sleep.yaml` -> Incorrect, not-valid (We should deny this creation request.)
`deployment/sleep-no-validation.yaml` -> Skip-validation (Based on `admission-webhook-example.qikqiak.com/validate: "false"` annotation, we skipped validation.)
`deployment/sleep-with-labels.yaml` -> Correct, valid (We should accept this creation request.)
```

### 7. A way of extend the operation event and function

In this demo, we only consider the Pod create request by specifying _operations_ at the ValidatingWebhookConfiguration's [matching request rules](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-rules) section in the [deployment/validatingwebhook.yaml](https://github.com/developer-guy/admission-webhook-example-with-openfaas/blob/master/deployment/validatingwebhook.yaml) file.

If we want to extend the operations, we can add a new [operation](https://github.com/kubernetes/kubernetes/blob/release-1.19/pkg/apis/admission/types.go#L158) for the Pods like _DELETE_, _UPDATE_, _CONNECT_ etc. By specifying a new operation, now apiserver being started to send a new event for this operation additional to create request.

Now we can specify more than one serverless function for the operation types by checking request operation type. For [example](https://github.com/developer-guy/admission-webhook-example-with-openfaas/blob/master/functions/requiredlabel/handler.go#L109): 

```go
switch req.Kind.Kind {
    case "Pod":
        switch req.Operation {
            case v1beta1.Create:
                // do something for create operation
            case v1beta1.Delete: 
                // do something for delete operation
        }
}
```

Also you can specify function name and namespace that you want to use while deploying the webhook server using these environment variables:

```yaml
 env:
   - name: FUNCTION_NAME
     value: requiredlabel
   - name: FUNCTION_NAMESPACE
     value: openfaas-fn
```

### Taking it further

* Mutating webhooks

You could take this example and convert it from validating webhooks to mutating webhooks. This is useful when a user wants to upgrade or modify objects that are created, such as adding which user created them, or adding a compulsory memory limit.

* Adding more functions

In the example I used a single function, however, you could register more than one function, so that you can then have a function for validating memory limits, and a separate one for checking that a minimum set of labels are present

### Join the community

Have you got questions, comments, or suggestions? Join the community on [Slack](https://slack.openfaas.io).

Would you like help to set up your OpenFaaS installation, or someone to call when things don't quite go to plan? [Our OpenFaaS PRO Subscription plan](https://www.openfaas.com/support/) gives you a say in the project roadmap, a support contact, and access to Enterprise-grade authentication with OIDC.

### Acknowledgements

* Special Thanks to [Alex Ellis](https://twitter.com/alexellisuk) for all guidance and for merging changes into OpenFaaS to better support this workflow.
* Special Thanks to [Furkan Türkal](https://twitter.com/furkanturkaI) for all the support.

### References

* [https://medium.com/ibm-cloud/diving-into-kubernetes-mutatingadmissionwebhook-6ef3c5695f74](https://medium.com/ibm-cloud/diving-into-kubernetes-mutatingadmissionwebhook-6ef3c5695f74)
* [https://blog.alexellis.io/get-started-with-openfaas-and-kind/](https://blog.alexellis.io/get-started-with-openfaas-and-kind/)
* [https://github.com/morvencao/kube-mutating-webhook-tutorial](https://github.com/morvencao/kube-mutating-webhook-tutorial)
* [https://github.com/developer-guy/admission-webhook-example-with-openfaas](https://github.com/developer-guy/admission-webhook-example-with-openfaas)
