# Kubernetes networking automation using GitOps

## Idea

Practice GitOps approach to manage network settings in Kubernetes cluster deployed locally.

## Tools

* [kind](https://kind.sigs.k8s.io/)
  * [Kind multi-node](https://docs.tigera.io/calico/latest/getting-started/kubernetes/kind)
  * [Kind quickstart](https://kind.sigs.k8s.io/docs/user/quick-start/)
  * [Kind networking](https://kind.sigs.k8s.io/docs/user/configuration/#networking)
* [Kustomize](https://kustomize.io/)
* [Calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/)
  * [Networking](https://docs.tigera.io/calico/latest/networking/)
  * [Security](https://docs.tigera.io/calico/latest/network-policy/)
* [Flux](https://fluxcd.io/)
  * [Ways of structuring your repositories](https://fluxcd.io/flux/guides/repository-structure/)
  * [Flux bootstrap](https://fluxcd.io/flux/installation/bootstrap/)
  * [``flux bootstrap github``](https://fluxcd.io/flux/cmd/flux_bootstrap_github/)
  * [``flux check``](https://fluxcd.io/flux/cmd/flux_check/)
  * [Manage Helm Releases](https://fluxcd.io/flux/guides/helmreleases/)
  * [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
* [Flagger](https://flagger.app/)
  * [Flagger Install with Flux](https://docs.flagger.app/install/flagger-install-with-flux)
  * [Flagger Install on Kubernetes](https://docs.flagger.app/install/flagger-install-on-kubernetes)
  * [NGINX Canary Deployments](https://docs.flagger.app/tutorials/nginx-progressive-delivery)
  * [Deployment Strategies](https://docs.flagger.app/usage/deployment-strategies)
  * [Canary deployment](https://docs.flagger.app/usage/how-it-works#canary-resource)
  * [Canary analysis with Prometheus Operator](https://docs.flagger.app/tutorials/prometheus-operator)
* Sample Cloud Native applications:
  * [Project pod tato Head](https://github.com/podtato-head/podtato-head)
  * [Podinfo](https://github.com/stefanprodan/podinfo)
* Terraform:
  * [How to GitOps Your Terraform](https://fluxcd.io/blog/2022/09/how-to-gitops-your-terraform/)
  * [Weave GitOps Terraform Controller - GitHub](https://github.com/weaveworks/tf-controller)
  * [Weave GitOps Terraform Controller - documentation](https://weaveworks.github.io/tf-controller/)
  * [An AWS IAM Policy Example for TF controller](https://github.com/tf-controller/aws-iam-policies)
  * [TF-controller demo: GitOps EKS nodegroup](https://github.com/tf-controller/eks-scaling)
  * [Flux provider](https://registry.terraform.io/providers/fluxcd/flux/latest)
  * [Kind provider](https://registry.terraform.io/providers/tehcyx/kind/latest)
* GitOps & Calico:
  * [GitOps and Admission Control for Calico Network Policy](https://medium.com/@bikramgupta/in-part-1-of-gitops-series-we-reviewed-the-need-for-using-gitops-for-calico-policies-and-how-to-660aa6b5e4c9)
  * [Enable GitOps for Kubernetes Security – Part 1](https://www.tigera.io/blog/enable-gitops-for-kubernetes-security-part-1/)
  * [Decentralized Calico Network Security Policy Deployment for GitOps – Part 2](https://www.tigera.io/blog/decentralized-calico-network-security-policy-deployment-for-gitops-part-2-2/)

## Problem

I need to automate network configuration in Kubernetes environment using GitOps approach. Moreover in order to use GitOps for IaC in hybrid environments, extend Kubernetes cluster by delivering controller for Terraform.

## Solution

In order to solve problems following actions needs to be taken:
- prepare lab with Kubernetes cluster to test whole flow
- use Calico as CNI plugin
- deploy Flux as GitOps tool
- prepare simple applications in the lab
- define network policies using Kustomize
- use Flagger for progressive delivery of changes
- extend cluster by deploying TF controller in order to allow applying IaC using GitOps

### Prepare local environment with Kubernetes cluster

Prepare configuration file:
```
cat > multi-node-k8s-no-cni.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF
```

Create K8s cluster:

```
kind create cluster --config multi-node-k8s-no-cni.yaml --name home-lab
```

Check nodes and cluster:

```
kubectl get nodes -o wide

kind get clusters
```

If necessary, remove cluster:

```
kind delete cluster -n home-lab
```

### Configure basic networking settings

#### [Calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/kind)

Install Tigera operator:

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

kubectl get pods -n tigera-operator
```

Download and create custom resource to configure Calico:

```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml -O

kubectl create -f custom-resources.yaml

watch kubectl get pods -n calico-system
```

#### [Cilium](https://github.com/cilium/cilium/tree/main/install/kubernetes/cilium)

```
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace=kube-system
```

#### [Flannel](https://github.com/flannel-io/flannel#deploying-flannel-manually)

Install Flannel into CNI-enabled Kubernetes:

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

#### [Weave Net](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#install)

Install Weave Net into CNI-enabled Kubernetes:

```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

### Prepare Flux to GitOps approach

Prepare classic personal access token and export GitHub PAT as an environment variable:

```
export GITHUB_TOKEN=***
```

Run pre-installation checks:

```
flux check --pre
```

Prepare folder for cluster:

```
mkdir -p clusters/home-lab
```

Flux bootstrap for GitHub:

```
flux bootstrap github \
  --token-auth \
  --owner=sebastianczech \
  --repository=k8s-networking-gitops \
  --branch=main \
  --path=clusters/home-lab \
  --personal
```

Check installation:

```
flux check
```

### Deploy demo application

#### podinfo

Define application:
- [clusters/home-lab/apps/kustomization.yaml](clusters/home-lab/apps/kustomization.yaml)
- [apps/podinfo/](apps/podinfo)

Reconcile changes in kustomization and check logs:

```
flux reconcile kustomization flux-system

flux logs --all-namespaces
```

Check ``podinfo`` application:

```
# kubectl -n podinfo exec podinfo-69d7bcd6c-qrz5v -it -- sh
~ $ curl podinfo:9898
{
  "hostname": "podinfo-69d7bcd6c-qrz5v",
  "version": "6.5.1",
  "revision": "0bc496456d884e00aa9eb859078620866aa65dd9",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v6.5.1",
  "goos": "linux",
  "goarch": "arm64",
  "runtime": "go1.21.1",
  "num_goroutine": "8",
  "num_cpu": "8"
}~ $
```

#### podtato-head

Define application:
- [clusters/home-lab/apps/kustomization.yaml](clusters/home-lab/apps/kustomization.yaml)
- [apps/podtato-head/](apps/podtato-head/)

Prepare NGINX ingress [network/ingress/](network/ingress/), which contains Helm repository and release.

Defined ingress in [apps/podtato-head/app.yaml](apps/podtato-head/app.yaml).

Describe ingress:

```
kubectl -n podtato describe ingress podtato-ingress
```

Verify ingress and check ``podtato-head`` application:

```
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot
tmp-shell> curl -H "Host: podtato.webapp.com" ingress-nginx-controller.ingress
```

Use in `curl` name of service and namespace `ingress`:

```
kubectl get svc -n ingress
```

### Use Kustomize to define Calico network policies

Before applying network polices, check access from default namespace:

```
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot

tmp-shell> nslookup podinfo.podinfo
tmp-shell> curl podinfo.podinfo.svc.cluster.local:9898
tmp-shell> curl podtato-head-entry.podtato:9000
```

Check if there are network policies:

```
calicoctl get networkPolicy --allow-version-mismatch
calicoctl get globalnetworkpolicy --allow-version-mismatch
```

Deploy [Calico Global Network Policy](network/calico/policies.yml) and check it:

```
calicoctl --allow-version-mismatch get globalNetworkPolicy deny-egress-default-ns -o yaml
```

Traffic to 1.1.1.1 should be blocked, other allowed:

```
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot

tmp-shell> curl --connect-timeout 1 1.1.1.1
curl: (28) Failed to connect to 1.1.1.1 port 80 after 1001 ms: Timeout was reached
tmp-shell> curl --connect-timeout 1 ifconfig.me
91.226.196.43
```

### Prepare Flagger for progressive delivery

Deploy:
- [Flagger for progressive delivery](common/flagger/)
- [Prometheus for monitoring](common/monitoring/)

Check Helm releases:

```
flux get helmreleases -n flagger-system
```

Debug issues with deploying Loadtester:

```
flux logs --all-namespaces --level=error
```

Deploy ingress for podinfo application using Kustomize and check it:

```
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot
tmp-shell> curl -H "Host: app.example.com" ingress-nginx-controller.ingress
```

Check values of metrics in Prometheus:

```
kubectl -n flagger-system port-forward services/flagger-prometheus 9090:9090
```

Generate traffic:

```
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -n podinfo
tmp-shell> while true; do curl -s -H 'Host: app.example.com' http://ingress-nginx-controller.ingress/version | jq .version; sleep 0.5; done
```

Create [canary deployment](apps/podinfo/canary.yaml):

```
kubectl -n podinfo get deployments,services,pod,canaries
kubectl -n podinfo describe canary podinfo
kubectl -n podinfo get canary/podinfo -oyaml | awk '/status/,0'
kubectl -n podinfo wait canary/podinfo --for=condition=promoted
```

[Change version of podinfo image](apps/podinfo/deployment.yaml) and observe progressive delivery:

```
watch kubectl -n podinfo describe canary podinfo
```

### Deploy IaC using Terraform and GitOps

Install TF-controller:

```
kubectl apply -f https://raw.githubusercontent.com/weaveworks/tf-controller/main/docs/release.yaml
```

Create AWS credentials and check them:

```
kubectl -n flux-system create secret generic aws-credentials \
    --from-literal=AWS_REGION=us-east-1 \
    --from-literal=AWS_ACCESS_KEY_ID=*** \
    --from-literal=AWS_SECRET_ACCESS_KEY=***

kubectl -n flux-system get secret aws-credentials -o jsonpath='{.data}'
```

Define [infrastructure](infrastructure) and check TF controller:

```
kubectl -n flux-system get terraform
kubectl -n flux-system describe terraform
```

### Use different approaches for GitOps

#### [ArgoCD](https://argo-cd.readthedocs.io/en/stable/getting_started/)

Install Argo CD:

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Download Argo CD CLI:

```
brew install argocd
```

Access the Argo CD server:

```
argocd admin initial-password -n argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Authenticate on https://localhost:8080 using generated credentials via web browser and CLI:

```
argocd login localhost:8080
```

Update password:

```
argocd account update-password
```

Create an application:

```
.........TODO.........
```

Sync (deploy) apps:

```
.........TODO.........
```
