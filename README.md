# JFrog Flux CD Test

This repository covers the installation and management of JFrog [Helm Charts](https://github.com/jfrog/charts) via [Flux CD](https://fluxcd.io/).

## Disclaimer
Flux CD is not officially supported by JFrog, this repository is for research purposes only.

Do not use the dummy master and join keys from this project in your environments.

## Intro
Our goal is to be able to deploy and manage JFrog products such as Artifactory and Xray through Flux CD, which is a GitOps tool. In this test, we will attempt to perform the following using Flux CD:
1. Install the Artifactory HA and Xray [Helm Charts](https://github.com/jfrog/charts) with custom values
2. Automatically apply values changes after performing a 'git push'
3. Upgrade the Artifactory/Xray application version

## Walkthrough
### Flux CD CLI
We will begin by installing the Flux CD CLI on our local machine. For Mac, Homebrew can be used:
```
brew install fluxcd/tap/flux
```
### GIT Setup
For our setup, we will need a Git repository. Flux CD supports various Git providers, but we'll be using GitHub. First, we'll need to generate a [GitHub Access Token](https://github.com/settings/tokens) and locally export it along with our username:
 ```
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
 ```
 Next, let's use the Flux CD CLI to check that our K8s cluster is compatible:
 ```
flux check --pre
 ```
We can now create a new repository in GitHub and install the Flux CD operator on our K8s cluster with a single command:
```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=helm-infra \
  --branch=main \
  --path=app-cluster \
  --personal
```
Finally, once the new Git repository named **helm-infra** is created, we'll need to clone it to our local machine:
```
git clone https://github.com/joeynaor/helm-infra
```
### Configuring HelmRepository
In order to pull JFrog charts, we need to configure the Official JFrog Helm repository as a Flux CD [HelmRepository object](https://fluxcd.io/docs/components/source/helmrepositories/):
```
flux create source helm jfrog \
--url https://charts.jfrog.io \
--interval 1m0s \
--export > helmrepo-jfrog.yaml
```
This command creates a local YAML file called **helmrepo-jfrog.yaml** which includes all the JFrog Helm repository information. The file should look like this:
```
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: jfrog
  namespace: flux-system
spec:
  interval: 1m0s
  url: https://charts.jfrog.io
```
We will place **helmrepo-jfrog.yaml** in our cloned repository under **helm-infra/app-cluster** and push the changes to Git:
```
git add -A && git commit -m "test" && git push
 ```
After a few moments, we'll be able to see the HelmRepository in Flux:
```
flux get sources helm

NAME 	READY	MESSAGE                                                                           	REVISION                                                        	SUSPENDED
jfrog	True 	Fetched revision: 39c3e634f20c387f09ad36c83882a5c16ff867f0ba5905ab9627bb1d63bac789	39c3e634f20c387f09ad36c83882a5c16ff867f0ba5905ab9627bb1d63bac789	False
```
### Deploying Artifactory as a HelmRelease
The [HelmRelease object](https://fluxcd.io/docs/components/helm/helmreleases/#disabling-resource-waiting) will be used to deploy a JFrog Helm chart, in our case, Artifactory. As before, we can create the initial YAML file using the Flux CD CLI:
```
flux create helmrelease artifactory \
  --source=HelmRepository/jfrog \
  --chart artifactory \
  --release-name artifactory \
  --chart-version 107.27.10 \
  --target-namespace default \
  --interval 1m0s \
  --export > helmrelease-artifactory.yaml
```
This will give us a basic HelmRelease YAML file called **helmrelease-artifactory.yaml** which is equivalent to deploying the Artifactory Chart with the default values.

We can some custom modifications such the chart-specific values (values.yaml) and allowing Flux CD to apply changes to the cluster even if pods are marked as not ready:
```
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: artifactory
  namespace: flux-system
spec:
  install:
    disableWait: true # don't wait for pod to be ready before applying changes
  upgrade:
    disableWait: true # don't wait for pod to be ready before applying changes
  interval: 1m0s
  releaseName: artifactory
  targetNamespace: default
  chart:
    spec:
      chart: artifactory
      version: 107.27.10
      sourceRef:
        kind: HelmRepository
        name: jfrog
  ##################################
  # Custom Artifactory values.yaml #
  ##################################
  values:
    artifactory:
      joinKey: 011b61f5c6a5928d4489df5fe6f120569c29c9e9dafb01f720c61853a3d90e2b
      masterKey: ce2dcb873c3094d5dd8e7a7136158e4f31866d802435306c1855a7213853ee34
      replicaCount: 2

    databaseUpgradeReady: true
    unifiedUpgradeAllowed: true

    postgresql:
      postgresqlPassword: password
  ##################################
```
Next, we place **helmrelease-artifactory.yaml** along with **helmrepo-jfrog.yaml** under **helm-infra/app-cluster** and push to Git once more:
```
git add -A && git commit -m "test" && git push
 ```
Flux CD should pick up the new commit and deploy Artifactory.
 
### Installing Xray
Deploying Xray (and any other chart/product) consists of the exact same logic as above. We will be using this HelmRelease YAML file called **helmrelease-xray.yaml**:
```
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: xray
  namespace: flux-system
spec:
  chart:
    spec:
      chart: xray
      version: 103.37.1
      sourceRef:
        kind: HelmRepository
        name: jfrog
  ###########################
  # Custom Xray values.yaml #
  ###########################
  values:
    rabbitmq:
      auth:
        password: f4PeLNdq25
    postgresql:
      postgresqlPassword: password
    xray:
      joinKey: 011b61f5c6a5928d4489df5fe6f120569c29c9e9dafb01f720c61853a3d90e2b
      masterKey: b46337c2a998d931246535fd2826928b83e3fb29bc4e5c8972c7c8e23f919b7c
      jfrogUrl: http://artifactory:8082
    unifiedUpgradeAllowed: true
    databaseUpgradeReady: true
  ###########################
  interval: 1m0s
  releaseName: xray
  targetNamespace: default
```
### Values, Changes and Upgrades
From this point onward, any changes done to our HelmRelease YAML files that were pushed to Git will trigger a redeploy the same way as Helm (helm upgrade). For instance, we can scale down the Artifactory deployment from 2 nodes to 1 by reconfiguring the replicaCount in **helmrelease-artifactory.yaml**:
```
  values:
    artifactory:
      joinKey: 011b61f5c6a5928d4489df5fe6f120569c29c9e9dafb01f720c61853a3d90e2b
      masterKey: ce2dcb873c3094d5dd8e7a7136158e4f31866d802435306c1855a7213853ee34
      replicaCount: 1
```
After pushing the changes to Git, Flux CD will scale the cluster down and remove the 2nd Artifactory pod.

Upgrading the Application version is very simple and consists of the same logic above. Simply change the value of  **chart.spec.version** in **helmrelease-artifactory.yaml**:
```
  chart:
    spec:
      chart: artifactory
      version: 107.29.7
```
Performing a Git push will trigger a rolling upgrade of all the Artifactory nodes in the cluster, one by one, from version 7.27.10 to 7.29.7.
