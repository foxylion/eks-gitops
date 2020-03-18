# EKS with git-ops

## Initial experimentations

``` bash
eksctl create cluster --name test --version 1.14 --region eu-central-1 --nodegroup-name default-workers \
  --node-type t3.medium --nodes 3 --managed

EKSCTL_EXPERIMENTAL=true \
eksctl enable repo --git-url git@github.com:foxylion/eks-gitops --git-email gitops@test.tld \
  --cluster test --region eu-central-1

EKSCTL_EXPERIMENTAL=true \
eksctl enable profile app-dev \
  --git-url git@github.com:foxylion/eks-gitops --git-email gitops@test.tld \
  --cluster test --region eu-central-1
```

## The right way, with config files

``` bash
eksctl create cluster -f eks-config/cluster.yml
```

### Deploy flux

Assuming you have a new, empty EKS cluster (see also [here](https://docs.fluxcd.io/en/1.18.0/tutorials/get-started.html))

``` 
# Deploy flux
kubectl create ns flux
fluxctl install --git-url git@github.com:foxylion/eks-gitops.git --git-email flux@example.tld \
   --git-path cluster --namespace flux | kubectl apply -f -

# Wait for successful deployment
kubectl -n flux rollout status -w deployment/flux

# Grant flux access to the git repository
fluxctl identity --k8s-fwd-ns flux

# Add key to github repository (with write access)
# https://github.com/foxylion/eks-gitops/settings/keys
```

### Grant IAM roles permission to the k8s API

* Under `cluster/base/authentication/` you'll find roles defined in Kubernetes
* Apply a modified version of `eks-config/aws-auth.yml` to grant IAM roles the permission for k8s roles

### Assign a pod an IAM role

* Add the IAM role to `eks-config/custer.yml` 

``` 
eksctl create iamserviceaccount -f eks-config/cluster.yml --approve
```

### Update nodegroups

* Change the name of modified nodegroups, modified nodegroups will then be created as a new nodegroup
* Afterwards we will delete all stale nodegroups, pods will be moved to the new nodegroups

``` bash
eksctl create nodegroup -f cluster.yml
eksctl delete nodegroup -f cluster.yml --only-missing
```

## Current issues

* Managed node groups in private subnets are currently not supported by eksctl - https://github.com/weaveworks/eksctl/issues/1575
* Do not mange the `aws-auth` config-map in the `kube-system` namespace through flux, or ensure that the role arn is correct (will differ between each cluster created, when wrong the cluster will go down)

## Helm templating

helm template -i flagger flagger/flagger --namespace flagger --set metricsServer=http://prometheus.monitoring:9090

## Prometheus

* By default the helm operator only selects ServiceMonitors with the label "Release: prometheus-operator". Use `prometheus.prometheusSpec.serviceMonitorSelector: {}` and `serviceMonitorSelectorNilUsesHelmValues: false` to select all ServiceMonitors
* Ingress Nginx controller needs to expose its metrics via a ServiceMonitor

## Things that could/should be improved

* Reduce the IAM policies attached to the nodes, use IAM roles attached to serviceaccounts instead (e.g.for cluster autoscaler, ... )

## Examples

### Run a pod with a service account that has a IAM role assigend

``` 
kubectl run -it --rm -n kube-system --image ubuntu:18.04 --serviceaccount external-dns shell -- /bin/bash
apt update
apt install curl unzip -y
curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
unzip awscliv2.zip
./aws/install

export AWS_PAGER=""
aws sts get-caller-identity
```

