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
