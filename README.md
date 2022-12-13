# scratch-on-rke2
This repo describes how to implement a scratch game compiled to html within a nginx web server running on a rk2 single node server cluster

## Prerequisites
You need a small Virtual Machine with Ubuntu Desktop 22.04 LTS.

## Overview
For deploying the game there are several task to do
- Installing a standard rke2 Single-Node-Clusters 
- Installing helm for the application deployment
- Installing the nginx Webserver-Deployments
- DNS Entry configuration (if you do not a have a DNS Server instead)
- Starting the Website

## rke2 Deployment
### Installing a standard rke2 Single-Node-Clusters 

```bash
export RKE2_VERSION="stable"
export KUBECONFIG="/etc/rancher/rke2/rke2.yaml"
export NODE_NAME=$(hostname -f)
export NODE_IP=$(ip -4 -o a | grep -i ens3 | tr -s ' ' | cut -d ' ' -f4 | cut -d '/' -f1)
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION="v${RKE2_VERSION}" sh -s - \
  --node-name="${NODE_NAME}" \
  --tls-san="${NODE_IP}" \
  --disable-cloud-controller \
  --write-kubeconfig-mode=0644
sudo systemctl enable rke2-server.service
sudo systemctl start rke2-server.service
sudo systemctl status rke2-server -l
sudo sed -i "s/127.0.0.1/${NODE_IP}/g" /etc/rancher/rke2/rke2.yaml
printf '\nKUBECONFIG="/etc/rancher/rke2/rke2.yaml"\n' | sudo tee -a /etc/environment
sudo tee /var/lib/rancher/rke2/server/manifests/traefik-config.yaml <<EOF
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: rke2-ingress-nginx
  namespace: kube-system
spec:
  valuesContent: |-
    ingressClass:
      enabled: true
      isDefaultClass: true
EOF

sudo systemctl restart rke2-server.service

```
## rke2 Testing and Configuration
```bash
sudo systemctl status rke2-server -l
/var/lib/rancher/rke2/bin/kubectl get nodes -A
/var/lib/rancher/rke2/bin/kubectl get addon -A
/var/lib/rancher/rke2/bin/kubectl get pods --all-namespace
echo "#####################"
echo "#####################"
echo "#####################"
echo "theKUBECONFIG variable must be set manually! Please execute the following command!"
echo "export KUBECONFIG="/etc/rancher/rke2/rke2.yaml" "

```
### Helm Installation
In order to install helm you can use the following script

```bash
export ARCH="amd64"
export ARCH_X=$(uname -m)
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
[[ ${ARCH_X} == "x86_64" ]] \
  && (export ARCH="amd64" && export ARCH_BIT="64bit") \
  || (export ARCH=${ARCH_X} && export ARCH_BIT=${ARCH_X})
export HELM_VERSION="3.10.0"
wget -O /tmp/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz \
  https://get.helm.sh/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz
pushd /tmp
wget -O helm-v${HELM_VERSION}-linux-${ARCH}.tmp \
  https://get.helm.sh/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz.sha256sum
cat helm-v${HELM_VERSION}-linux-${ARCH}.tmp | grep helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz > helm-v${HELM_VERSION}-linux-${ARCH}.sha256sum
[[ "$(sha256sum -c helm-v${HELM_VERSION}-linux-${ARCH}.sha256sum)" == *"OK" ]] || exit 1
rm -rf helm-v${HELM_VERSION}-linux-${ARCH}.tmp \
       helm-v${HELM_VERSION}-linux-${ARCH}.sha256sum
popd
sudo tar -zxvf /tmp/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz \
  --strip-components=1 \
  -C /usr/local/bin \
  linux-${ARCH}/helm
rm -rf /tmp/helm-v${HELM_VERSION}-linux-${ARCH}.tar.gz
sudo chown root:root /usr/local/bin/helm
sudo chmod 755 /usr/local/bin/helm
sudo ln -s /usr/local/bin/helm /usr/local/bin/helm3
helm list --all-namespaces
```
### Installing the nginx Webserver-Deployments
For the deployment we are using the standard public nginx helm charts from bitnami. You can optionally fetch the whole chart data from the repository.
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install merrychristmas bitnami/nginx \
  --namespace merrychristmas \
  --create-namespace \
  --timeout 600s \
  --set global.storageClass="localPath" \
  --set cloneStaticSiteFromGit.enabled=true \
  --set cloneStaticSiteFromGit.repository="https://github.com/cmeyer29/scratch-on-rke2.git" \
  --set cloneStaticSiteFromGit.branch="main" \
  --set cloneStaticSiteFromGit.interval=3600 \
  --set service.type="ClusterIP" \
  --set ingress.enabled=true \
  --set ingress.hostname="merrychristmas.toyou" \
  --set ingress.path="/"  \
  --set ingress.ingressClassName="rke2-ingress-nginx"
```

### DNS Configuration on the host
you need to set a dns entry in the hosts data /etc/hosts 
```bash
sudo sed -i '/^127.0.0.1/ s/$/ merrychristmas.toyou /' /etc/hosts
```

### DNS Configuration within the CoreDNS ConfigMap
You need the set an entry within the CoreDNS ConfigMap for a functioning DNS Resolution
```bash
/var/lib/rancher/rke2/bin/kubectl edit cm -n kube-system rke2-coredns-rke2-coredns
```
### Reloading the CoreDNS ConfigMap
in order to reload the changed ConfigMap we have to delete the current CoreDNS pod
```bash
/var/lib/rancher/rke2/bin/kubectl delete pods -n kube-system $(/var/lib/rancher/rke2/bin/kubectl get pods -n kube-system | grep -i rke2-coredns-rke2-coredns | cut -d' ' -f1)
```

### Testing
Testing via Curl
```bash
curl -k http://merrychristmas.toyou
```
### Starting the site 
```bash
firefox --new-window http://merrychristmas.toyou
```

<img src="https://github.com/cmeyer29/scratch-on-k3s/blob/main/images/christmas-game.jpg" width="600"/>

### Deinstall nginx
```bash
helm uninstall merrychristmas -n merrychristmas 
```
### Deinstall rke2
```bash
/usr/bin/rke2-uninstall.sh
```

### Sources

- [rke2 Installation](https://docs.rke2.io/)
- [helm Installation](https://helm.sh/docs/helm/helm_install/)
- [Bitnami Nginx Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/nginx/)
- [Scratch 3.0](https://scratch.mit.edu/)
