# Kubernetes v1.28 SidecarContainers - API for sidecar containers
As per https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#api-for-sidecar-containers, K8S now allows you to 
specify a restartPolicy for init containers which is independent of the Pod and other init containers.

This features is available starting with Kubernetes 1.28 in alpha through a feature gate named SidecarContainers.
Alpha features are NOT enabled by default.

Through the years, I have been researching different ways to capture traffic in pods (see references).

# How to enable the feature gate
I compiled a small step-by-step install for a self-managed k8s setup
- Create 3 ubuntu nodes
- On the control-plane node:
```
curl https://raw.githubusercontent.com/xxradar/k8s-calico-oss-install-containerd/main/setup-cluster-config.sh | bash
```
- On the worker nodes:
```
curl https://raw.githubusercontent.com/xxradar/install_k8s_ubuntu/main/setup_node_latest.sh | bash
```
- Run the join command on the worker nodes
```
sudo kubeadm join 10.11.2.231:6443 --token eow8gw.8863eelhollpn37p \
    --discovery-token-ca-cert-hash sha256:1e0ec482fcee39edbfzxxzxz5e6a7e57217bd1e57c23e2d318ef7e16759947e
```
## Install a CNI
```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
```
sudo snap install helm --classic

helm repo add cilium https://helm.cilium.io/

helm install cilium cilium/cilium --version 1.14.0 \
    --namespace kube-system \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set encryption.enabled=true \
    --set encryption.type=wireguard \
    --set kube-proxy-replacement=strict \
    --set ingressController.enabled=partial \
    --set ingressController.loadbalancerMode=shared \
    --set ingressController.service.type="NodePort" \
    --set loadBalancer.l7.backend=envoy
```
