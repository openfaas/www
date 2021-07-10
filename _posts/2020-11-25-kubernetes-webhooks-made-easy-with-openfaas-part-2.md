---
title: "Writing Kubernetes Admission Controllers with OpenFaaS"
description: "Learn how to write your Kubernetes Admission Controllers with OpenFaaS functions"
date: 2020-11-25
image: /images/2020-10-27-k8s-validatingwebhook-openfaas/puzzle.jpg
categories:
  - arkade
  - kubectl
  - faas-cli
  - admissionwebhooks
  - mutatingadmissionwebhooks
  - k8s extensibility
author_staff_member: developer-guy
dark_background: true
---

Learn how to write your Kubernetes Admission Controllers with OpenFaaS functions

<p align="center">
<img height="128" src="/images/openfaas/kubernetes.png">
</p>

## Introduction to Kubernetes Admission Controllers
In my last [post](https://www.openfaas.com/blog/kubernetes-webhooks-made-easy-with-openfaas/), we talked about what Kubernetes Admission Webhooks are and how can we use Validating Admission Webhook.

Today we will explore how to build a Kubernetes Admission Controller with a function.

The main difference between the two types of admission webhook is validating webhooks can reject a request, but they cannot modify the object they are receiving in the admission request, while mutating webhooks can modify objects by creating a patch that will be sent back in the admission response and can reject a request too. If a webhook rejects a request, an error is returned to the end-user.

Also, the other difference between the two types are validating webhooks are called in parallel and mutating webhooks are called in-sequence by the api-server.

## Use-cases
Multi-tenant Kubernetes creates some special challenges when it comes to resource sharing and security.Multi-tenant Kubernetes is a Kubernetes deployment where multiple applications or workloads run side-by-side.When it comes to resource sharing, you must ensure that each tenant has access to the resources they need.Requests and limits are the mechanisms Kubernetes uses to control resources such as CPU and memory.So, if you forget to set a limits&requests for your application, you may affect the cluster state bad and this causes to disturb the other tenants in the platform.Because when you forget to set limits&request for the application, application is going to try to consume all the resources available on the node.To avoid that, we are going to create a Mutating Admission Webhook, this webhook will check your deployment for the resources available, If you forgot to set resources it will inject the default resources for you automatically.

## The Scenario
We have a deployment with no resources defined.We are going to try to create this deployment and we want to make sure that this webhook will not let us to do that.

There is one important thing we need to say about Mutating Admission webhook is that:

> Mutating Admission webhooks, mutations are performed via JSON patches. While the JSON patch standard includes a lot of intricacies that go well beyond the scope of this discussion, the Go data structure in our example as well as its usage should give the user a good initial overview of how JSON patches work:

```golang
type patchOperation struct {
  Op    string      `json:"op"`
  Path  string      `json:"path"`
  Value interface{} `json:"value,omitempty"`
}
```

For setting the field .spec.securityContext.runAsNonRoot of a pod to true, we construct the following patchOperation object:

```golang
patches = append(patches, patchOperation{
  Op:    "add",
  Path:  "/spec/securityContext/runAsNonRoot",
  Value: true,
})
```

Reminder, we will use the same workflow defined in previous post like below:

* Kubernetes API -> Webhook (w/TLS) -> OpenFaaS Gateway (w/HTTP) -> OpenFaaS Function

![Workflow](/images/2020-10-27-k8s-validatingwebhook-openfaas/admission-controller-phases.png)
> Credit: [https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/)

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

```sh
# Forward the gateway to your machine
kubectl rollout status -n openfaas deploy/gateway
kubectl port-forward -n openfaas svc/gateway 8080:8080 &

# If basic auth is enabled, you can now log into your gateway:
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)
echo -n $PASSWORD | faas-cli login --username admin --password-stdin
```

You can access them again at any time with `arkade info openfaas`

### 3. Clone the project

* Clone the sample from GitHub

```sh
$ git clone https://github.com/developer-guy/admission-webhook-example-with-openfaas
$ cd admission-webhook-example-with-openfaas
$ git checkout feature/mutating
```

* Let's explore the structure of the project.

```
deployment/ --> includes necessary manifests and scripts for the deployment of the project
functions/ --> includes templates and the requiredlabel function itself
Dockerfile --> includes instructions to build an image of the project
build --> automated way to build and push an image of the project
```

### 4. Deploy MutatingAdmissionWebhook

* We will generate the TLS certificates required for the ValidatingAdmissionWebhook using the following: [Kubernetes TLS Certificates Management](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)

```sh
$ cd deployment
$ sh webhook-create-signed-cert.sh
```

* Get the Certificate Authority (CA) from the local cluster

```sh
$ export CA_BUNDLE=$(kubectl config view --minify --flatten -o json | jq -r '.clusters[] | select(.name == "'$(kubectl config current-context)'") | .cluster."certificate-authority-data"')
$ sed -e "s|\${CA_BUNDLE}|${CA_BUNDLE}|g" mutatingwebhook.yaml | kubectl apply -f -
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
# Label the default namespace to enable the webhook
$ kubectl label namespaces default admission-webhook-example=enabled
```

* This time we are using "fileinjector" as a FUNCTION_NAME environment variable in the deployment.yaml file

```yaml
env:
  - name: FUNCTION_NAME
    value: fileinjector
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
$ faas-cli new fileinjector --lang go-middleware
$ cd fileinjector
$ go mod init fileinjector
$ # fill the handler.go with the corresponding code: [functions/fileinjector/handler.go](https://github.com/developer-guy/admission-webhook-example-with-openfaas/blob/feature/mutating/functions/fileinjector/handler.go)
$ go get
```

* Deploy the function

```sh
$ cd functions
$ faas-cli up -f fileinjector.yml --build-arg GO111MODULE=on # (build-push-deploy) make sure you are using your docker hub username. i.e: devopps
```

* Verify the functions that are working in `openfaas-fn` namespace

```sh
$ kubectl get pods --namespace openfaas-fn
```

### 6. Testing the whole workflow

* First, check the [deployment](https://github.com/developer-guy/admission-webhook-example-with-openfaas/blob/feature/mutating/deployment/busybox.yaml) manifest and see there is no resources defined for the container.

```sh
apiersion: apps/v1
kind: Deployment
metadata:
  name: busybox
  namespace: default
  labels:
    app: busybox
spec:
  selector:
    matchLabels:
      app: busybox
  replicas: 1
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
        - name: busybox
          image: busybox
          command: [ "/bin/sh" ]
          args: [ "-c", "while true; do cat /etc/config/hello-openfaas.txt; sleep 2; done" ]
```

* Then, you can create the deployment by applying this manifest file. After doing this, you will notice that container resources are injected automatically by the mutating webhook with the default values defined by the mutating webhook.

```sh
$ kubectl apply -f busybox.yaml
```

* Check the YAML of the Deployment, you will see the default resources.

```sh
$ kubectl get deployments busybox -ojsonpath='{.spec.template.spec.containers[?(@.name=="busybox")].resources}'
{"limits":{"cpu":"125Mi","memory":"75Mi"},"requests":{"cpu":"100Mi","memory":"50Mi"}}
```


## 7. What can you do with it ?

As it stands, the demo is not ready for production environment but it gives you an overview about how you can create a mutating webhook , how you can enforce some kind of best practices(inject default resources in this case) for your cluster with the few lines of code.You can extend this demo for your organizational needs as you wish.

### Join the community

Have you got questions, comments, or suggestions? Join the community on [Slack](https://slack.openfaas.io).

Would you like help to set up your OpenFaaS installation, or someone to call when things don't quite go to plan? [Our Premium Subscription plan](https://www.openfaas.com/support/) gives you a say in the project roadmap, a support contact, and access to Enterprise-grade authentication with OIDC.

### Acknowledgements

* Special Thanks to [Alex Ellis](https://twitter.com/alexellisuk) for all guidance and for merging changes into OpenFaaS to better support this workflow.

* Special Thanks to [Furkan Türkal](https://twitter.com/furkanturkaI) for all the support.

### References
* [https://banzaicloud.com/blog/k8s-admission-webhooks](https://banzaicloud.com/blog/k8s-admission-webhooks)
* [https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/)
* [https://medium.com/ibm-cloud/diving-into-kubernetes-mutatingadmissionwebhook-6ef3c5695f74](https://medium.com/ibm-cloud/diving-into-kubernetes-mutatingadmissionwebhook-6ef3c5695f74)
* [https://blog.alexellis.io/get-started-with-openfaas-and-kind/](https://blog.alexellis.io/get-started-with-openfaas-and-kind/)
* [https://github.com/morvencao/kube-mutating-webhook-tutorial](https://github.com/morvencao/kube-mutating-webhook-tutorial)
* [https://github.com/developer-guy/admission-webhook-example-with-openfaas](https://github.com/developer-guy/admission-webhook-example-with-openfaas)
* [https://platform9.com/blog/kubernetes-multi-tenancy-best-practices/](https://platform9.com/blog/kubernetes-multi-tenancy-best-practices/)
