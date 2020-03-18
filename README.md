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
kubectl -n flux rollout status deployment/flux

# Grant flux access to the git repository
fluxctl identity --k8s-fwd-ns flux

# Add key to github repository (with write access)
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

## Helm templating

helm template -i flagger flagger/flagger --namespace flagger --set metricsServer=http://prometheus.monitoring:9090

## Prometheus

* By default the helm operator only selects ServiceMonitors with the label "Release: prometheus-operator". Use `prometheus.prometheusSpec.serviceMonitorSelector: {}` and `serviceMonitorSelectorNilUsesHelmValues: false` to select all ServiceMonitors
* Ingress Nginx controller needs to expose its metrics via a ServiceMonitor

