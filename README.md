
# Setup OpenShift platform from scratch

## Overview

This setup supposes that following choices have been made towards deployment on K8S:
* Use of HELM (Tiller) for deployment
* FluxCD to satisfy GitOps needs

![gitops](https://github.com/fluxcd/helm-operator-get-started/blob/master/diagrams/flux-helm-operator-registry.png)


### Prerequisites

* Brew (for Mac)
* VirtualBox

In case you run into trouble with installation of VirtualBox on Mac use the following commands (boot into recovery)

```
$ spctl kext-consent disable
$ spctl kext-consent add VB5E2TV963
$ spctl kext-consent enable 
$ reboot
```

### Setup OpenShfit locally

From here on commands have been setup to be executed with Mac, adaption for Windows may be required.

```
$ brew cask install minishift
```

The --vm-driver virtualbox flag will need to be given on the command line each time the minishift start command is run. For example:
```
$ minishift start --vm-driver virtualbox
```

Setting the vm-driver option as a persistent configuration option allows you to run minishift start without explicitly passing the --vm-driver virtualbox flag each time. You may set the vm-driver persistent configuration option as follows:
```
$ minishift config set vm-driver virtualbox
```


<!--## Setup Flux with HELM

### Install CLI

Here are the most important ones
```
$ brew install kubernetes-helm
$ brew install kubectl-cli
$ brew install openshift-cli
```

### Initialize Helm client

Ensure tiller runs in client-mode only as the tiller server will run on OC anyway:
```
$ helm init --client-only
```

Export PATH variable
```
$ export TILLER_NAMESPACE=tiller
```

### Install the Tiller server

Login as administrator
```
oc login -u system:admin
```

Create a new project:
```
$ oc new-project tiller
```

In principle this can be done using helm init, but currently the helm client doesn’t fully set up the service account rolebindings that OpenShift expects. To try to keep things simple, we’ll use a pre-prepared OpenShift template instead. The template sets up a dedicated service account for the Tiller server, gives it the necessary permissions, then deploys a Tiller pod that runs under the newly created SA.

```
$ oc process -f https://github.com/openshift/origin/raw/master/examples/helm/tiller-template.yaml -p TILLER_NAMESPACE="${TILLER_NAMESPACE}" -p HELM_VERSION=v2.16.0 | oc create -f -

serviceaccount/tiller created
role.authorization.openshift.io/tiller created
rolebinding.authorization.openshift.io/tiller created
deployment.extensions/tiller created

```

At this point, you’ll need to wait for a moment until the Tiller server is up and running:
```
$ oc rollout status deployment tiller

deployment "tiller" successfully rolled out
```

We’ll check that the Helm client and Tiller server are able to communicate correctly by running
```
$ helm version

Client: &version.Version{SemVer:"v2.16.0", GitCommit:"...", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.16.0", GitCommit:"...", GitTreeState:"clean"}
```

Note: This is a quick guide and by no means a production ready Tiller setup, please look into ‘![Securing your Helm installation](https://helm.sh/docs/using_helm/#securing-your-helm-installation)’ before promoting to production.

### Setup namespaces

In this example we'll use a simple setup of one namespace for only 1 environment, ```dev```, ```stg```, ```prd```. By no means does this represent a production worthy setup.

Repeat steps below for each environment:

```
$ oc new-project dev
```

```
$ oc policy add-role-to-user edit "system:serviceaccount:${TILLER_NAMESPACE}:tiller"

clusterrole.rbac.authorization.k8s.io/edit added: "system:serviceaccount:tiller:tiller"
```

Add a ```developer``` or other account to project
```
$ oc adm policy add-role-to-user edit developer
```

Removing this role is as easy as when we've added it

```
$ oc adm policy remove-role-from-user edit
```

### Setup Flux

Apply the Helm Release CRD:
```
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flux/helm-0.10.1/deploy-helm/flux-helm-release-crd.yaml
```

Add the Flux chart repo:
```
helm repo add fluxcd https://charts.fluxcd.io
```

Install Flux and its Helm Operator by specifying your fork URL:
```
helm install --name flux \
--set rbac.create=true \
--set helmOperator.create=true \
--set git.url=git@github.com:tpx01/city-service-config \
--namespace flux \
fluxcd/flux
```

FAIL-->

## Setup FLUX

### Install CLI

Here are the most important ones
```
brew install fluxctl
```

### Setup Flux

Login as administrator
```
oc login -u system:admin
```
<!--
oc create clusterrolebinding clusterrole.rbac \
  --clusterrole=cluster-admin --user=fluxsa

Create a flux service account
```
$ oc create sa fluxsa
$ oc describe sa fluxsa
$ oc adm policy --as system:admin add-cluster-role-to-user cluster-admin fluxsa
```-->

By default the 'developer' user (nor 'admin' user for that matter) have create permissions for namespaces nor manage the cluster.
```
oc adm policy --as system:admin add-cluster-role-to-user cluster-admin developer
```

Create a flux namespace from which workloads are tracked
```
kubectl create ns flux
```

```
export GHUSER="TPX01"
fluxctl install \
--git-user=${GHUSER} \
--git-email=${GHUSER}@users.noreply.github.com \
--git-url=git@github.com:${GHUSER}/city-service-config \
--git-path=namespaces,workloads \
--namespace=flux | kubectl apply -f -
```

Wait for Flux to start:
```
$ kubectl -n flux rollout status deployment/flux

deployment "flux" successfully rolled out
```

At startup Flux generates a SSH key and logs the public key. Find the SSH public key by installing fluxctl and running:
```
$ fluxctl identity --k8s-fwd-ns flux

ssh-rsa ...
```

In order to sync your cluster state with git you need to copy the public key and create a deploy key with write access on your GitHub repository.

Open GitHub, navigate to your fork, go to Setting > Deploy keys, click on Add deploy key, give it a Title, check Allow write access, paste the Flux public key and click Add key. See the GitHub docs for more info on how to manage deploy keys.


### Force sync

If you don't want to wait for next poll on configuration repository, you can easily tickle Flux to pull changes.

```
$ fluxctl sync --k8s-fwd-ns flux

Synchronizing with git@github.com:TPX01/city-service-config
Revision of master to apply is aaae9d9
Waiting for aaae9d9 to be applied ...
Done.
```

## Fancy dashboarding

https://www.weave.works/docs/scope/latest/installing/#ose


To install Weave Scope on OpenShift, you first need to login as system:admin user with the following command:

```
oc login -u system:admin
```

Next, create a dedicated project for Weave Scope then apply policy changes needed to run Weave Scope:

```
$ oc new-project weave
$ oc adm policy add-role-to-user edit developer
$ oc adm policy add-cluster-role-to-user cluster-admin -z weave-scope
$ oc adm policy add-scc-to-user privileged -z weave-scope
$ oc adm policy add-scc-to-user anyuid -z default
````

The installation method for Scope on OpenShift is very similar to the one described above for Kubernetes, but instead of kubectl apply ... you need to use oc apply ... and install it into the namespace of the weave project you have just created, and not the weave namespace, i.e.:

```
$ oc apply -f 'https://cloud.weave.works/k8s/scope.yaml'
```

To access the Scope app from the browser, please refer to Kubernetes instructions above.

```
$ kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

