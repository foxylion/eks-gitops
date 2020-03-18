# EKS with git-ops

## (Intitally) Manual Process

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

## With config files

``` bash
eksctl create cluster -f cluster.yml
eksctl update cluster -f cluster.yml
```

### Connect to flux

Assuming you have a new, empty EKS cluster (see also [here](https://docs.fluxcd.io/en/1.18.0/tutorials/get-started.html))

``` 
# Deploy flux
kubectl create ns flux
fluxctl install --git-url git@github.com:foxylion/eks-gitops.git --git-email flux@example.tld \
   --namespace flux | kubectl apply -f -

# Wait for successful deployment
kubectl -n flux rollout status deployment/flux

# Grant flux access to the git repository
fluxctl identity --k8s-fwd-ns flux

# Add key to github repository (with write access)
```

### Update nodegroups

Change the name of modified nodegroups, they will be created, afterwards delete the old nodgroups

``` bash
eksctl create nodegroup -f cluster.yml
eksctl delete nodegroup -f cluster.yml --only-missing
```

## Current issues

* Managed node groups in private subnets are not supported by eksctl - https://github.com/weaveworks/eksctl/issues/1575

## Helm templating

helm template -i flagger flagger/flagger --namespace flagger --set metricsServer=http://prometheus.monitoring:9090

## Prometheus

* By default the helm operator only selects ServiceMonitors with the label "Release: prometheus-operator". Use `prometheus.prometheusSpec.serviceMonitorSelector: {}` and `serviceMonitorSelectorNilUsesHelmValues: false` to select all ServiceMonitors
* Ingress Nginx controller needs to expose its metrics via a ServiceMonitor

