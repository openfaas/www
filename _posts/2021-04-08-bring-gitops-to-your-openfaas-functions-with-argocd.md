---
title: "Bring GitOps to your OpenFaaS functions with ArgoCD"
description: "In this post you'll learn how to manage OpenFaaS functions based on GitOps principles by using ArgoCD"
date: 2021-04-15
image: /images/2021-04-08-bring-gitops-to-your-openfaas-functions-with-argocd/argo-horizontal.jpg
categories:
- arkade
- kubectl
- argocd
- gitops
- openfaas-operator
- helm
author_staff_member: batuhan
dark_background: true

---

In this post you'll learn how to manage OpenFaaS functions based on GitOps principles by using ArgoCD

## What is GitOps?

[Alexis Richardson](https://twitter.com/monadic) coined the term _GitOps_ in 2017. Since then it's gained significant interest in the enterprise companies because it allows for easier auditing of deployments.

With a GitOps approach, build artifacts from your Continuous Integration (CI) pipeline are deployed to your cluster. The desired state is set in a number of configuration files in a Git repository, and a second agent component runs inside the cluster to compare the delta. The differences are known as drift, and the agent's role is to detect and correct it.

![Separation of CI and CD](https://images.contentstack.io/v3/assets/blt300387d93dabf50e/blt31fc28ca69fa30a8/5a5e940703361d7a0b2936a5/gitops_cd_pipeline.png)

> From [Weaveworks.com](https://www.weave.works/technologies/gitops/)

> In the diagram above, we can see that GitOps separates our code repository, where our teams push and test code, from the repo containing deployment information, aka "config repo". When new images or artifacts are created, the tags and versions in the config repo can be updated automatically or via a separate commit from a team member.

Two of the most popular open-source projects for GitOps are [Flux](https://fluxcd.io/), which was created at Weaveworks. Intuit, an American payroll company created [ArgoCD](https://argoproj.github.io/argo-cd/). Both projects were donated to the Cloud Computing Foundation (CNCF) to encourage broader use and contributions.

We have [previously looked at FluxCD with OpenFaaS](https://www.openfaas.com/blog/openfaas-flux/), but in this post we'll show how to use ArgoCD with the OpenFaaS Function CRD to continuously deploy your functions.

You can read more about ArgoCD on the [project homepage](https://argoproj.github.io/argo-cd/).

## Tutorial

In this section, we are going to create two application for ArgoCD, first one is OpenFaaS Operator, the second one is the repository that holds OpenFaaS functions manifest files.

ArgoCD can work against Kubernetes manifests in a various ways including kustomize, ksonnet, jsonnet, as well as other plugins which can generate YAML files that Kubernetes can apply. Today, we will be using a Helm chart and YAML manifests.

We will use the OpenFaaS [Helm Chart](https://github.com/openfaas/faas-netes) to deploy OpenFaaS and its Operator. Once OpenFaaS Operator is deployed, we can use the _Custom Resource_ called _"Function"_ in order to define our OpenFaaS functions like any other Kubernetes resources such as Deployment, Pod etc.

In the second part, we'll to use plain manifests to deploy OpenFaaS functions. A dedicated chart can also be created for deploying functions.

### Conceptual architecture
 
![Conceptual architecture](/images/2021-04-08-bring-gitops-to-your-openfaas-functions-with-argocd/conceptual.jpg)

> Conceptual architecture: ArgoCD monitoring two repositories and deploying OpenFaaS along with a set of functions

### A recap on the OpenFaaS Operator

We need to use the Operator, because Argo can only apply Kubernetes YAML files, and cannot use the OpenFaaS REST API at this time. The normal installation of OpenFaaS uses its REST API to create functions using the CLI, REST API or UI. It also has a mode called "operator mode" where a Custom Resource can be used with `kubectl` to apply and deploy functions.

```yaml
apiVersion: openfaas.com/v1
kind: Function
metadata:
  name: nodeinfo
  namespace: openfaas-fn
spec:
  name: nodeinfo
  image: functions/nodeinfo:latest
```

The `faas-cli generate` command can be used to convert the OpenFaaS stack.yml file into the Function CustomResource for use with `kubectl`.

The _OpenFaaS Operator_ comes with an extension to the Kubernetes API that allows you to manage OpenFaaS functions in a
declarative manner. The operator implements a control loop that tries to match the desired state of your OpenFaaS
functions, defined as a collection of custom resources, with the actual state of your cluster.

To get more detail please refer to this [link](https://blog.alexellis.io/introducing-the-openfaas-operator/).

### Prerequisites

Once we have arkade, we can create a cluster and install ArgoCD. If you prefer, you can also manually download all the tools required, and find the instructions for ArgoCD's helm chart.

 * [arkade](https://get-arkade.dev) (v0.7.13) Kubernetes marketplace

  ```bash
  # Run with or without sudo
  $ curl -sLS https://dl.get-arkade.dev | sudo sh
  ```

* KinD (Kubernetes in Docker) v0.10.0

  Kubernetes is our recommendation for teams running at scale, but in this demo we will be
    using [KinD](https://kind.sigs.k8s.io/docs/user/quick-start/) for the sake of simplicity.

  ```bash
  $ arkade get kind --version=v0.10.0
  ```

* kubectl v1.21.0

  You can control your cluster using [kubectl](https://github.com/kubernetes/kubectl) CLI.

  ```bash
  $ arkade get kubectl --version=v1.21.0
  ```

* ArgoCD CLI v2.0.0

  ArgoCD CLI controls an Argo CD server. More detailed installation instructions can be found via
    the [CLI installation documentation](https://argoproj.github.io/argo-cd/cli_installation/). Fortunately, we can
    install it via arkade too.

  ```bash
  $ arkade get argocd --version=v2.0.0
  ```

### Setup

### 1. Provision a local Kubernetes Cluster with KinD

You can start a Kubernetes cluster with KinD if you don't have one already.

```bash
$ kind create cluster
```

Verify if your cluster working properly before moving onto the next step.

```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:49809
KubeDNS is running at https://127.0.0.1:49809/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### 2. Deploy ArgoCD

There are various ways to install ArgoCD, one of them is _ArgoCD Operator_, and the other one is just with plain YAML
manifest. We are going to deploy ArgoCD with plain YAML manifest in this section.

arkade is not only for the CLI tooling, it also helps you to get started to install applications using helm under the hood, 
hopefully arkade also supports installing ArgoCD.

```bash
$ arkade install argocd
Using Kubeconfig: /Users/batuhan.apaydin/.kube/config
Node architecture: "amd64"
=======================================================================
= ArgoCD has been installed                                           =
=======================================================================


# Get the ArgoCD CLI
arkade install argocd

# Port-forward the ArgoCD API server
kubectl port-forward svc/argocd-server -n argocd 8443:443 &

# Get the password
PASS=$(kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d)
echo $PASS

# Or log in:
argocd login --name local 127.0.0.1:8443 --insecure \
 --username admin \
 --password $PASS

# Open the UI:
https://127.0.0.1:8443

# Get started with ArgoCD at
# https://argoproj.github.io/argo-cd/#quick-start

Thanks for using arkade!

```

Verify if everything is working properly in _argocd_ namespace before moving onto the next step.

```bash
$ kubectl get pods --namespace argocd
NAME                                  READY   STATUS    RESTARTS   AGE
argocd-application-controller-0       1/1     Running   0          86s
argocd-dex-server-5dd657bd9-zkhb8     1/1     Running   0          86s
argocd-redis-759b6bc7f4-92l4b         1/1     Running   0          86s
argocd-repo-server-6c495f858f-rx8hh   1/1     Running   0          86s
argocd-server-859b4b5578-sczvk        1/1     Running   0          86s
```

### 3. Deploy OpenFaaS Operator and OpenFaaS functions through ArgoCD

There are two ways of defining applications to ArgoCD, one of them is by using CLI, the other one is by using CRDs. We
are going to demonstrate both of them in here.

#### Option 1: by using CLI

First, we need to authenticate to ArgoCD server.Argo CD v1.9. Later the initial password for the admin account is
auto-generated and stored as clear text in the field password in a secret named argocd-initial-admin-secret in your Argo
CD installation namespace.

Because ArgoCD Server is running on a Kubernetes, we should do port-forwarding first in order to access the server, it
can be achieved by the following command easily:

```bash
# Port-forward the ArgoCD API server
$ kubectl port-forward svc/argocd-server -n argocd 8443:443 &
Forwarding from 127.0.0.1:8443 -> 443
Forwarding from [::1]:8443 -> 443
```

You can simply retrieve this password using kubectl:

```bash
# Get the password
$ PASS=$(kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d)
echo $PASS
```

After that, by using the username _admin_ and the password from above, lets log into ArgoCD:

```bash
# log in:
$ argocd login --name local 127.0.0.1:8443 --insecure \
 --username admin \
 --password $PASS
WARNING: server certificate had error: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
Username: admin
Password:
'admin:login' logged in successfully
Context 'local' updated


# Open the UI:
$ open https://127.0.0.1:8443

# Get started with ArgoCD at
# https://argoproj.github.io/argo-cd/#quick-start
```

To get more detail about login process, please refer to
this [link](https://argoproj.github.io/argo-cd/getting_started/#4-login-using-the-cli).

Verify if you log into ArgoCD properly before moving on the next step.

```bash
$ argocd account list
NAME   ENABLED  CAPABILITIES
admin  true     login

$ argocd app list
NAME  CLUSTER  NAMESPACE  PROJECT  STATUS  HEALTH  SYNCPOLICY  CONDITIONS  REPO  PATH  TARGET

```

After you logged into ArgoCD, lets create our applications.

```bash
$ kubectl create namespace openfaas
namespace/openfaas created

$ kubectl create namespace openfaas-fn
namespace/openfaas-fn created

$ argocd app create openfaas-operator \
      --repo https://github.com/openfaas/faas-netes.git \
      --dest-namespace openfaas \
      --dest-server https://kubernetes.default.svc \
      --path chart/openfaas \
      --helm-set operator.create=true \
      --helm-set generateBasicAuth=true \
      --helm-set functionNamespace=openfaas-fn \
      --self-heal \
      --sync-policy automatic
application 'openfaas-operator' created
```

> When deploying internally (to the same cluster that Argo CD is running in), https://kubernetes.default.svc should be used as the application's K8s API server address. To get more detail about registering another cluster to deploy apps to, please refer to this [link](https://argoproj.github.io/argo-cd/getting_started/#5-register-a-cluster-to-deploy-apps-to-optional).

Verify if everything is working before moving onto the next step.

First, check the application status:

```bash
$  argocd app list -o wide
NAME               CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                        PATH            TARGET
openfaas-operator  https://kubernetes.default.svc  openfaas   default  Synced  Healthy  Auto        <none>      https://github.com/openfaas/faas-netes.git  chart/openfaas
```

Then, check the _openfaas_ namespace for the installation:

```bash
$ kubectl get pods --namespace openfaas
NAME                                 READY   STATUS    RESTARTS   AGE
alertmanager-84d7b88f74-kt7jg        1/1     Running   0          4m6s
basic-auth-plugin-85d885557c-cgz4j   1/1     Running   0          4m6s
gateway-764b7b4865-p4rk8             2/2     Running   1          4m6s
nats-5fdf6476f-pmftg                 1/1     Running   0          4m6s
prometheus-844fffb-5rkc7             1/1     Running   0          4m6s
queue-worker-7cbc9f8688-qfww6        1/1     Running   0          4m6s
```

Before deploying OpenFaaS functions as custom resources, check the available CRDs on the Kubernetes, you should see the
list like the following:

```bash
$ kubectl get customresourcedefinitions.apiextensions.k8s.io
NAME                       CREATED AT
applications.argoproj.io   2021-04-09T09:33:59Z
appprojects.argoproj.io    2021-04-09T09:33:59Z
functions.openfaas.com     2021-04-09T10:43:03Z
profiles.openfaas.com      2021-04-09T10:43:03Z
```

If you see the same output above, it means you are now ready to deploy OpenFaaS functions.

I prepared a repository that holds the function definitions, so, before running the following code, don't forget to fork
your own copy of this repository and give it to _"--repo"_ flag.

```bash
$ argocd app create openfaas-functions \
      --repo https://github.com/developer-guy/bring-gitops-to-your-openfaas-functions-with-argocd.git \
      --dest-server https://kubernetes.default.svc \
      --path functions \
      --self-heal \
      --sync-policy automatic
application 'openfaas-functions' created
```

Verify if everything is working before moving onto the next step.

First, check the application status, now, you should see two applications:

```bash
$  argocd app list -o wide
NAME                CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                                                                      PATH            TARGET
openfaas-functions  https://kubernetes.default.svc             default  Synced  Healthy  Auto        <none>      https://github.com/developer-guy/bring-gitops-to-your-openfaas-functions-with-argocd.git  functions
openfaas-operator   https://kubernetes.default.svc  openfaas   default  Synced  Healthy  Auto        <none>      https://github.com/openfaas/faas-netes.git
```

Then, check the _openfaas-fn_ namespace for the installation:

```bash
$ kubectl get functions --namespace openfaas-fn
NAME       AGE
nodeinfo   35s
```

Verify if it is working.

```
$ kubectl get pods --namespace openfaas-fn
NAME                        READY   STATUS    RESTARTS   AGE
nodeinfo-6d5434f4744-l6kfd   1/1     Running   0          2m6s
```

#### Option 2: by using CRDs

We are going to apply _**apps-of-apps**_ pattern in here, which means we can create an app that creates other apps,
which in turn can create other apps. This allows you to declaratively manage a group of app that can be deployed and
configured in concert. In order to do that, we create an _apps-of-apps.yaml_ in the repository and within this manifest
we refer to the exact path which holds the actual application manifests.

To get more detail about _**apps-of-apps**_ pattern, please refer to
this [link](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/#app-of-apps).

Lets deploy the applications.

```bash
$ kubectl create namespace openfaas
namespace/openfaas created

$ kubectl create namespace openfaas-fn
namespace/openfaas-fn created

$ argocd app create apps-of-apps \
      --repo https://github.com/developer-guy/bring-gitops-to-your-openfaas-functions-with-argocd.git \
      --dest-server https://kubernetes.default.svc \
      --path . \
      --self-heal \
      --sync-policy automatic
application 'apps-of-apps' created
```

We just created one application called _apps-of-apps_ in here, but if you look at the output of the _list_ command, you
should see three application:

```bash
$  argocd app list
NAME                CLUSTER                         NAMESPACE  PROJECT  STATUS     HEALTH   SYNCPOLICY  CONDITIONS  REPO                                                                                      PATH            TARGET
apps-of-apps        https://kubernetes.default.svc             default  OutOfSync  Healthy  Auto        <none>      https://github.com/developer-guy/bring-gitops-to-your-openfaas-functions-with-argocd.git  applications
openfaas-functions  https://kubernetes.default.svc             default  Synced     Healthy  Auto        <none>      https://github.com/developer-guy/bring-gitops-to-your-openfaas-functions-with-argocd.git  functions
openfaas-operator   https://kubernetes.default.svc  openfaas   default  OutOfSync  Missing  Auto        <none>      https://github.com/openfaas/faas-netes.git                                                chart/openfaas
```

Verify if everything is working.

```bash
$ kubectl get functions --namespace openfaas-fn
NAME       AGE
nodeinfo   35s

$ kubectl get pods --namespace openfaas-fn
NAME                        READY   STATUS    RESTARTS   AGE
nodeinfo-7c564f4744-l6kfd   1/1     Running   0          101s
```

### Taking it further

### Join the community

Have you got questions, comments, or suggestions? Join the community on [Slack](https://slack.openfaas.io).

Would you like help to set up your OpenFaaS installation, or someone to call when things don't quite go to
plan? [Our OpenFaaS PRO Subscription plan](https://www.openfaas.com/support/) gives you a say in the project roadmap, a
support contact, and access to Enterprise-grade authentication with OIDC.

### Acknowledgements

* [Alex Ellis](https://twitter.com/alexellisuk) for guidance, editing and providing the diagrams.
* [Furkan Türkal](https://twitter.com/furkanturkaI) for all the support.

### References

* [Argo CD - Declarative GitOps CD for Kubernetes](https://argoproj.github.io/argo-cd/getting_started)
* [Getting Started with the OpenFaaS Kubernetes Operator](https://dzone.com/articles/getting-started-with-the-openfaas-kubernetes-opera)
* [Weaveworks' Guide To GitOps](https://www.weave.works/technologies/gitops/)
* [GitOps](https://www.gitops.tech)
