# istio-workshop

Kubernetes Istio Workshop

* Istio
* Prometheus
* Jaeger
* Kiali

## Prerequisites

* Azure or Google Cloud
* kubectl
* helm

## Azure Setup

> **NB!** The following commands are for BASH or any other POSIX compliant shell
> and may not work directly in CMD nor PowerShell. We recommend using the Azure
> Cloud Shell.

Make sure you have the latest `az` installed

```
az --version
```

Create a new resource group

```
az group create --name istio-workshop -l westeurope
```

Creeate new AKS cluster

```
az aks create \
  --name istio-cluster \
  --resource-group istio-workshop \
  --node-count 1 \
  --node-vm-size Standard_B12ms \
  --enable-addons monitoring \
  --generate-ssh-keys
```

Get the AKS specific resource group

```
export AKS_RG=$(az aks show --resource-group istio-workshop --name istio-cluster --query nodeResourceGroup -o tsv)
echo $AKS_RG
```

Reserve a new public IP

```
az network public-ip create --name istio-gateway --resource-group $AKS_RG --allocation-method Static
```

Get the address of the static IP

```
export ISTIO_IP=$(az network public-ip show --name istio-gateway --resource-group $AKS_RG -o tsv --query "ipAddress")
echo $ISTIO_IP
```

Get the kubectl config for the AKS cluster

> **NB!** This will overwrite the `~/.kube/config` if you have an existing
> kubectl config

```
az aks get-credentials --resource-group istio-workshop -a --name istio-cluster --file - > ~/.kube/config
```

Verify the kubectl setup by running the following

```
kubectl version
```

## Install Istio

This will allow Helm and Tiller to manage the lifecycle of Istio.

Before you continue, make sure you have cloned this repository locally:

```
git clone --recurse-submodules https://github.com/evry-bergen/istio-workshop.git
```

Make sure you have a service account with the cluster-admin role defined for Tiller. If not already defined, create one using following command:

```
kubectl apply -f istio/install/kubernetes/helm/helm-service-account.yaml
```

Install Tiller on your cluster with the service account:

```
helm init --service-account tiller
```

Install the istio-init chart to bootstrap all the Istio’s CRDs:

```
helm install istio/install/kubernetes/helm/istio-init --name istio-init --namespace istio-system --wait
```

Verify that all 23 Istio CRDs were committed to the Kubernetes api-server using the following command:

```
kubectl get crds | grep 'istio.io' | wc -l
```

Install the istio chart to install and set up Istio and surrounding tooling:

```
helm upgrade istio istio/install/kubernetes/helm/istio --wait --install \
    --namespace istio-system \
    --values istio/install/kubernetes/helm/istio/values-istio-demo-common.yaml \
    --values istio/install/kubernetes/helm/istio/values-istio-demo.yaml \
    --set gateways.istio-ingressgateway.loadBalancerIP=$ISTIO_IP
```

Verify that the Kubernetes services corresponding to your selected profile have been deployed.

```
kubectl get svc -n istio-system
```

Ensure the corresponding Kubernetes pods are deployed and have a STATUS of `Running`:

```
kubectl get pods -n istio-system
```

Install Istio Gateways in order to access telemetry services remotely:

```
kubectl apply -f gateways
```

You should now be able to access the Telemetry components:

* Kiali http://`$ISTIO_IP`:15029
* Prometheus http://`$ISTIO_IP`:15030
* Grafana http://`$ISTIO_IP`:15031
* Tracing http://`$ISTIO_IP`:15032

## Bookinfo Application

* https://istio.io/docs/examples/bookinfo/

## Reference

* https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough
* https://docs.microsoft.com/en-us/azure/aks/static-ip
* https://docs.microsoft.com/en-us/azure/aks/istio-install
* https://istio.io/docs/setup/install/helm/
* https://istio.io/docs/tasks/telemetry/gateways/
