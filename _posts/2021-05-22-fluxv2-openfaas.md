---
title: "See how Flux's Git Ops ToolKit can simplify delivery"
description: "See how Flux can simplify serverless deployments to OpenFaaS"
date: 2021-05-22    
image: /images/2021-05-22-flux/earth.jpg
categories:
- arkade
- kubectl
- gitops
- openfaas-operator
- flux
- CI/CD
author_staff_member: alistair
dark_background: true

---

We learn how to manage OpenFaaS and install functions using Flux.

# What is Flux
Flux is an open source project by [weaveworks](https://www.weave.works/oss/flux/) which enables 
application delivery using GitOps. Your cluster config is stored in Git and the flux controllers continually work to ensure
that the cluster config matches the state in Git. This allows us to take the "cattle not pets" mentality applied to modern cloud computing further and apply the same thinking
to our clusters. 

We no longer need to manually issue `helm install` or `kubectl apply` commands to manage our clusters and 
configuration changes are audited and easily reviewed as Git PullRequests.

![flux v2 diagram](/images/2021-05-22-flux/gitops-toolkit.png)
*image from [Flux](https://github.com/fluxcd/flux2/blob/main/docs/_files/gitops-toolkit.png)*
 
# GitOps

`GitOps` is a set of practices that aim to democratise our "Operations" type practices such as deploying new application 
versions or installing new software on servers that takes practices used in development such as using Git as a `single source of truth`
and building auditability and accountability through peer review on Pull Requests to manage our day-to-day operations activities.

I won't go into much mode detail as there are entire conferences dedicated to this topic and a few paragraphs from me won't 
do this topic justice! Take some time to familiarise yourself with the concept if it is new to you. 

### Assumptions

* You have a Github Account (any git server can be used, please see section on installing flux for a link to the flux docs)


## Tools
We are going to use `arkade` to download our CLIs, you can install it like this

```bash
curl -SLs https://dl.get-arkade.dev | sh

# For MacOS and Linux users:
sudo mv arkade /usr/local/bin/

# For Git Bash users:
mv arkade /usr/bin/
```

Now get `kubectl`, `faas-cli`, `flux` and `kind` for use later.


```bash
arkade get kubectl
arkade get kind
arkade get faas-cli
arkade get flux
```

follow the instructions in the output from the commands above to add the tools to your `$PATH`

# Get a Kubernetes cluster

Head on over to your favourite `Kubernetes as a Service` provider, we are using [linode](https://www.linode.com/) (An OpenFaaS Homepage sponsor)
to provide us with our cluster. 

You can also use `kind`, `k3d` or `minikube` to provision a cluster locally too!

### Kubernetes on Linode
1. Create an account on Linode, [this is the signup page](https://login.linode.com/signup). New users get $100 credit (at time of writing)

2. Head to the `kubernetes` tab on the sidebar

3. Create a cluster in a region close to you and select at least 3 nodes with 1cpu and 2GB ram. This should be enough for 
our applications.
   
4. Download the `kubeconfig` once the cluster has been created and set the `KUBECONFIG` environment variable

In my case, this is how to set the environment variable.
```sh 
export KUBECONFIG=~/Downloads/kubeconfig
```

### Kubernetes locally with `kind`

There is nothing stopping you using a local cluster on your laptop. With just a few commands you can spin up a local cluster instead. 
You wont be able to expose your functions on the internet without some extra software like [inlets](https://inlets.dev) to provide a public IP.

Create a `kind` cluster abd wait for it to be ready.

```sh
kind create cluster \
  --name flux-openfaas
  
kind export kubeconfig --name flux-openfaas 
```

# Install Flux
Flux uses a CLI to setup and install the flux controllers into the cluster. Once we have the CLI we can 
install the controllers and then start adding resources into our cluster! We installed the `flux1 cli` earlier, so we are ready to go.

We have the Flux cli we need to `bootstrap` the cluster and create a git repository. 

For this section I'm using Github, you can find instructions for other Git providers [in the flux docs](https://fluxcd.io/docs/installation/#bootstrap)

Generate a `github personal access token` with `repo` permissions [like this](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token)

Export this as an environment variable

```sh 
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your github username>
```

Run the bootstrap for a repository on your personal GitHub account:

```sh 
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=openfaas-flux \
  --path=clusters/linode \
  --personal
```

After a short time we should see the flux controllers installed into the `flux-system` namespace!

```sh 
$ kubectl get pods -n flux-system
NAME                                       READY   STATUS    RESTARTS   AGE
helm-controller-85bfd4959d-w5ccr           1/1     Running   0          38s
kustomize-controller-6977b8cdd4-gblq7      1/1     Running   0          38s
notification-controller-5c4d48f476-7tdkk   1/1     Running   0          38s
source-controller-85fb864746-gzrxq         1/1     Running   0          38s
```

This step has:
* Installed the `flux controllers` into your cluster 
* Created a github repository, named `openfaas-flux` (unless you changed that). 
* Added a `Deploy key` to Github with access to that repository

These flux controllers are using the deploy key to read changes in the github repository and install things based on 
changes! In the next section we will add some `yaml` manifests into the git repository and see Flux install these into the cluster.

# Install OpenFaaS

Clone your git repository, for me the commands were

```sh 
git clone git@github.com:$GITHUB_USER/openfaas-flux.git
cd openfaas-flux/clusters/linode
```

Our flux controllers are syncing manifests we change in the `clusters/linode` directory that we specified as the path on isntall.

We can now add a `HelmRepository` and `HelmRelease` to install the OpenFaaS Chart

You can get more information on the [flux components in the docs](https://fluxcd.io/docs/components/)

Create our OpenFaaS namespaces by adding a file with these contents to the repository called `openfaas-namespaces.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openfaas
  labels:
    role: openfaas-system
    access: openfaas-system
---
apiVersion: v1
kind: Namespace
metadata:
  name: openfaas-fn
  labels:
    role: openfaas-fn
```

Add a `HelmRepository` which tells Flux where it can find the OpenFaaS helm chart
```sh 
flux create source helm openfaas \ 
  --url https://openfaas.github.io/faas-netes/ \
  --export > openfaas-helmchart.yaml
```

We need to make some changes to the default OpenFaaS chart installation, we are going to enable the `openfaas operator` 
which allows us to manage our functions using the `Functions CRD` 

Create a file called `values.yaml` and add these lines

```yaml
operator:
  create: true
  
generateBasicAuth: true
```

Finally add a `HelmRelease` which uses the `HelmRepository` as the source of the helm chart.
```sh 
flux create helmrelease openfaas \ 
  --source=HelmRepository/openfaas \ 
  --chart openfaas --values=values.yaml \ 
  --target-namespace=openfaas \
  --export > openfaas-helmrelease.yaml
```

Commit these files into your git repository and watch OpenFaaS get installed into your cluster! Don't add the `values.yaml` as flux will try to install this as a kubernetes object!
```sh 
git add openfaas*
git commit -m "Install openfaas into the cluster"
git push
```

After a few seconds you should see openfaas pods start to be created in the `openfaas` namespace.

```sh 
$ kubectl get pods -n openfaas
NAME                                READY   STATUS    RESTARTS   AGE
alertmanager-6457d4c9-ptbqp         1/1     Running   0          31s
basic-auth-plugin-b9cd7fb66-6gm8c   1/1     Running   0          31s
gateway-dbfb888df-m75vz             2/2     Running   0          30s
nats-6b6564d858-sxtwn               1/1     Running   0          31s
prometheus-788d6cdd47-54dhl         1/1     Running   0          31s
queue-worker-5f6cb648db-8vbjg       1/1     Running   0          31s
```

### Troubleshooting
Flux manages things through the use of CRDs, they usually have a good description of things that are not working,
so below are some handy commands to check on the resources we created above if anything isn't working.
```sh 
# Check if flux was able to install your manifests
# If this shows an error then you need to fix it! its usually formatting issues
kubectl get kustomization -n flux-system

# Check if we could download the helm chart for openfaas
# If this doesn't show "ready=True" then check the HelmRepository is correct
kubectl get helmchart -n flux-system

# To check the HelmRepository
kubectl get helmrepository -n flux-system

# Finally, check the HelmRelease, if this shows errors then you need to work out how to fix!
kubectl describe helmrelease -n flux-system openfaas
```

# Deploy our first Function
We are now going to generate a `Function` CRD which tells OpenFaaS to deploy a function, for speed we are going to use a
pre-built store function as an example. You can use your existing functions or build new ones if you prefer.

> to generate Function CRD files for your existing `stack.yaml` functions you can use `faas-cli generate --yaml stack.yaml`

```sh 
faas-cli generate --namespace openfaas-fn --from-store nodeinfo > nodeinfo.yaml
faas-cli generate --namespace openfaas-fn --from-store figlet > figlet.yaml
```

Add these files to git and push 

```sh 
git add figlet.yaml nodeinfo.yaml
git commit -m "Add openfaas functions"
git push
```

You should see flux pull the latest git revision and install these CRDs.

```sh 
$ kubectl get functions
NAME       AGE
figlet     89s
nodeinfo   89s

$ kubectl get pods -n openfaas-fn
NAME                        READY   STATUS    RESTARTS   AGE
figlet-56475b7d8-8xwcq      1/1     Running   0          27s
nodeinfo-6b54b47769-g7gkk   1/1     Running   0          27s
```

# Update our function
To see the power of flux and GitOps in action we can delete the `functions` CRDs from our cluster and 
watch as Flux puts them right back for us. 

```sh 
$ kubectl delete functions -n openfaas-fn --all
function.openfaas.com "figlet" deleted
function.openfaas.com "nodeinfo" deleted


$ kubectl get functions -n openfaas-fn
No resources found in openfaas-fn namespace.

# Reconcile our flux kustomization, for speed, but this will happen
# automatically every 10 minutes (which is configurable)

$ flux reconcile kustomization flux-system
► annotating Kustomization flux-system in flux-system namespace
✔ Kustomization annotated
◎ waiting for Kustomization reconciliation
✔ Kustomization reconciliation completed
✔ applied revision main/8dd6da7a8deaec21d3c7036f35caaaebf547177b

$ kubectl get functions -n openfaas-fn
NAME       AGE
figlet     37s
nodeinfo   37s
```

The functions were re-added by flux, it recognised that the objects that existed in Git didn't exist in the 
cluster and added them back.

As you can see, there is no more wondering if you applied the latest changes to a cluster, go and check Git
and the Flux CRD objects to see what git revision was applied and if it succeeded.

# GitOps and CI/CD

We have seen how we can use Git as our "single source of truth" about the state of our cluster and how GitOps tools like `flux` 
can keep our cluster in-sync with our config.


## Further reading

We have written about GitOps and Flux before on the OpenFaaS Blog.

- [Manage your OpenFaaS functions with ArgoCD](https://www.openfaas.com/blog/bring-gitops-to-your-openfaas-functions-with-argocd/)
- [Applying GitOps to OpenFaaS with Flux Helm Operator](https://www.openfaas.com/blog/openfaas-flux/)