# minikube local development environment

This project deals with the installation and configuration of minikube for local development purposes.

## Requirements

* Running version of Docker 19+
* `kubectl` binary in PATH 

## Install minikube binary

Grab latest minikube [release](https://github.com/kubernetes/minikube/releases) suitable for your OS. At the time of writing this was 1.8.2. 

## Create minikube cluster


```
minikube start \
    --driver=docker \
    --addons=dashboard \
    --addons=ingress \
    --addons=metrics-server \
    --addons=storage-provisioner \
    --addons=ingress-dns \
    --addons=logviewer \
    --keep-context=true \
    --cpus=6 \
    --memory=8g \
    --disk-size=50g \
    --mount=true \
    --bootstrapper=kubeadm \
    --extra-config=kubelet.authentication-token-webhook=true \
    --extra-config=kubelet.authorization-mode=Webhook \
    --extra-config=scheduler.address=0.0.0.0 \
    --extra-config=controller-manager.address=0.0.0.0 \
    --extra-config=kubelet.read-only-port=10255 \
    --kubernetes-version=1.14.7
```

## Install monitoring tools

The manifests have been generated from jsonnet definitions provided by [@carlosedp](https://github.com/carlosedp/cluster-monitoring). 

Following command has to be executed twice to solve a race condition problem with Kubernetes. It's due to the fact that it defines Custom Resource Definitions and tries to use them immediately after to start Prometheus, AlertManager and Grafana.

```
kubectl --context minikube apply -f cluster-monitoring/
sleep 5
kubectl --context minikube apply -f cluster-monitoring/
```

Give it a minute or so for various components to pull images and start up. Eventually observe `monitoring` namespace in Kubernetes Dashboard (available by executing `minikube dashboard`) until all is green. 

Note: `smtp-server` deployment will fail to start if a Secret with username and password for SMTP relay is not provided. If you don't intend to send emails from Grafana or other apps then keep it scaled down to 0 replicas. Else read about proper configuration of the service at [carlosedp/docker-smtp](https://github.com/carlosedp/docker-smtp) and scale it up to 1 after. 

Once Prometheus and Grafana boot up you can access them at:

* http://grafana-172-17-0-2.nip.io (no auth required)
* http://prometheus-172-17-0-2.nip.io (admin:admin)
* http://alertmanager-172-17-0-2.nip.io (no auth required)

## Install Calico

Calico is what will make NetworkPolicies work. These are firewalling rules for Pods similar to how security groups in AWS work.

```
kubectl --context minikube apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## Installalation and configuration of ArgoCD

Create `argocd` namespace and install latest ArgoCD after

```
kubectl --context minikube create namespace argocd
kubectl --context minikube apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Creating ArgoCD Ingress

Before you create the Ingress resource make sure nginx-ingress controller is running with SSL passthrough enabled. Add `- '--enable-ssl-passthrough'` in the list of `args`.

```
kubectl --context minikube -n kube-system edit deployments.apps nginx-ingress-controller
```

Then create the Ingress

```
cat <<EOF | kubectl --context minikube apply -f -
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: 'true'
    nginx.ingress.kubernetes.io/ssl-passthrough: 'true'
spec:
  rules:
    - host: argocd-172-17-0-2.nip.io
      http:
        paths:
          - path: /
            backend:
              serviceName: argocd-server
              servicePort: https
EOF
```

### Download ArgoCD CLI

Open [ArgoCD UI](https://argocd-172-17-0-2.nip.io) in your browser. Default username is `admin` and the password is set to the name of the Pod that runs ArgoCD Server. You can obtain it by running following command.

```
kubectl --context minikube get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

Once in navigate to the [Help](https://argocd-172-17-0-2.nip.io/help) section and download ArgoCD Client suitable for your OS.

```
curl -o argocd -k https://argocd-172-17-0-2.nip.io/download/argocd-linux-amd64
chmod +x argocd
```

### Login to ArgoCD Server from CLI

```
argocd login argocd-172-17-0-2.nip.io --username admin --password $(kubectl --context minikube get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2)
```

### Change admin password

Previously we've mentioned that the initial password is set to the name of the Pod that runs ArgoCD server. This is going to change if for all the reasons the Deployment starts a new Pod. Hence it's recommended to change the password to something easier for you to remember. Note: Change `deadbeef00` with something you like.

```
argocd account update-password --current-password $(kubectl --context minikube get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2) --new-password deadbeef00
```

### Allow ArgoCD to manage minikube cluster

```
argocd cluster add minikube --in-cluster
```

### Add repo in ArgoCD

```
argocd repo add https://stefanprodan.github.io/podinfo --type helm --name podinfo
```

or if you use a private repo

```
argocd repo add https://github.com/private/repo.git --username $USERNAME --password $PASSWORD
```

Remember to replace `$USERNAME` and `$PASSWORD` with correct credentials for your usecase.

### Create first ArgoCD application

```
argocd app create hello-world \
    --repo https://stefanprodan.github.io/podinfo \
    --helm-chart podinfo \
    --revision 3.2.0 \
    --dest-namespace default \
    --dest-server https://kubernetes.default.svc \
    --helm-set ingress.enabled=true \
    --helm-set ingress.path=/ \
    --helm-set 'ingress.hosts[0]=helloworld-172-17-0-2.nip.io' \
    --helm-set replicaCount=2
```

### Sync ArgoCD application

```
argocd app sync hello-world
```

### Wait for the application to be deployed

```
argocd app wait hello-world
```

### Verify application has been deployed

```
~ # argocd app list
NAME         CLUSTER                         NAMESPACE  PROJECT  STATUS  HEALTH   SYNCPOLICY  CONDITIONS  REPO                                    PATH  TARGET
hello-world  https://kubernetes.default.svc  default    default  Synced  Healthy  <none>      <none>      https://stefanprodan.github.io/podinfo        3.2.0
```

## Install Octant (Alternative Kubernetes Dashboard and much more ;))

Download suitable binary for your OS from Octant [repository](https://github.com/vmware-tanzu/octant/releases/latest/). Run `octant` from CLI after. Remember to authenticate (AWS MFA etc.) before running the application else it won't be able to connect to secured Kubernetes contexts. Octant is an application that runs locally on your system and as such makes Kubernetes cluster management easier in case Dashboard hasn't been installed.