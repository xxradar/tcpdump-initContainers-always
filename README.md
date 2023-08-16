# Kubernetes v1.28 SidecarContainers - API for sidecar containers
As per https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#api-for-sidecar-containers, K8S now allows you to 
specify a restartPolicy for init containers which is independent of the Pod and other init containers.

This features is available starting with Kubernetes 1.28 in alpha through a feature gate named SidecarContainers.
Alpha features are NOT enabled by default.

Through the years, I have been researching different ways to capture traffic in pods (see references).

## How to enable the feature gate
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
- Run the join command on the worker nodes (see output on control-plane node)
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
If all works out fine:
```
kubectl get no
NAME            STATUS   ROLES           AGE   VERSION
ip-10-1-2-153   Ready    <none>          42m   v1.28.0
ip-10-1-2-194   Ready    control-plane   43m   v1.28.0
ip-10-1-2-36    Ready    <none>          42m   v1.28.0
```
## Run tcpdump as a SidecarContainer (init container)
```
cat >tcpdumper.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mynginx
  labels:
    app: mynginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mynginx
  template:
    metadata:
      labels:
        app: mynginx
    spec:
      containers:
        - name: mynginx
          image: nginx:latest
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: tcpdumper
          image: xxradar/hackon
          restartPolicy: Always
          command: ['bash', '-c', 'tcpdump -w /opt/logs.pcap']
          volumeMounts:
            - name: data
              mountPath: /opt
      volumes:
        - name: data
          emptyDir: {}
EOF
```
```
kubectl apply -f tcpdumper.yaml
```
```
kubectl get po
NAME                       READY   STATUS             RESTARTS      AGE
...
mynginx-5c7565866b-flfct   2/2     Running            0             35m
...
```## Testing
```
pod_ip=$(kubectl get po -o json  -l app=mynginx | jq -r .items[].status.podIP)
```
```
kubectl run --image xxradar/hackon --env pod_ip=$pod_ip curler -- curl $pod_ip
```
```
kubectl exec -it -c tcpdumper $(kubectl get po -o name -l app=mynginx) -- tcpdump -r /opt/logs.pcap
```
```
reading from file /opt/logs.pcap, link-type EN10MB (Ethernet), snapshot length 262144
13:52:43.643270 IP6 fe80::98a8:75ff:fe4f:1767 > ff02::16: HBH ICMP6, multicast listener report v2, 1 group record(s), length 28
13:52:43.643326 IP6 fe80::98a8:75ff:fe4f:1767 > ff02::2: ICMP6, router solicitation, length 16
13:52:43.675308 IP6 fe80::64b7:e8ff:fe60:3800 > ff02::16: HBH ICMP6, multicast listener report v2, 1 group record(s), length 28
13:52:43.675321 IP6 fe80::64b7:e8ff:fe60:3800 > ff02::2: ICMP6, router solicitation, length 16
13:52:44.411291 IP6 fe80::98a8:75ff:fe4f:1767 > ff02::16: HBH ICMP6, multicast listener report v2, 1 group record(s), length 28
13:52:44.635322 IP6 fe80::64b7:e8ff:fe60:3800 > ff02::16: HBH ICMP6, multicast listener report v2, 1 group record(s), length 28
13:52:47.899260 IP6 fe80::98a8:75ff:fe4f:1767 > ff02::2: ICMP6, router solicitation, length 16
13:52:47.899254 IP6 fe80::64b7:e8ff:fe60:3800 > ff02::2: ICMP6, router solicitation, length 16
13:52:55.370193 IP ip-10-0-2-140.eu-west-3.compute.internal.52600 > mynginx-5c7565866b-flfct.http: Flags [S], seq 783507533, win 62167, options [mss 8881,sackOK,TS val 1503500973 ecr 0,nop,wscale 7], length 0
13:52:55.370218 ARP, Request who-has ip-10-0-0-56.eu-west-3.compute.internal tell mynginx-5c7565866b-flfct, length 28
13:52:55.370223 ARP, Reply ip-10-0-0-56.eu-west-3.compute.internal is-at 66:b7:e8:60:38:00 (oui Unknown), length 28
13:52:55.370226 IP mynginx-5c7565866b-flfct.http > ip-10-0-2-140.eu-west-3.compute.internal.52600: Flags [S.], seq 3470911100, ack 783507534, win 62083, options [mss 8881,sackOK,TS val 1488507083 ecr 1503500973,nop,wscale 7], length 0
13:52:55.370852 IP ip-10-0-2-140.eu-west-3.compute.internal.52600 > mynginx-5c7565866b-flfct.http: Flags [.], ack 1, win 486, options [nop,nop,TS val 1503500975 ecr 1488507083], length 0
13:52:55.370928 IP ip-10-0-2-140.eu-west-3.compute.internal.52600 > mynginx-5c7565866b-flfct.http: Flags [P.], seq 1:75, ack 1, win 486, options [nop,nop,TS val 1503500975 ecr 1488507083], length 74: HTTP: GET / HTTP/1.1
13:52:55.370938 IP mynginx-5c7565866b-flfct.http > ip-10-0-2-140.eu-west-3.compute.internal.52600: Flags [.], ack 75, win 485, options [nop,nop,TS val 1488507083 ecr 1503500975], length 0
13:52:55.371142 IP mynginx-5c7565866b-flfct.http > ip-10-0-2-140.eu-west-3.compute.internal.52600: Flags [P.], seq 1:239, ack 75, win 485, options [nop,nop,TS val 1488507083 ecr 1503500975], length 238: HTTP: HTTP/1.1 200 OK
13:52:55.371254 IP mynginx-5c7565866b-flfct.http > ip-10-0-2-140.eu-west-3.compute.internal.52600: Flags [P.], seq 239:854, ack 75, win 485, options [nop,nop,TS val 1488507084 ecr 1503500975], length 615: HTTP
13:52:55.371637 IP ip-10-0-2-140.eu-west-3.compute.internal.52600 > mynginx-5c7565866b-flfct.http: Flags [.], ack 239, win 485, options [nop,nop,TS val 1503500976 ecr 1488507083], length 0
13:52:55.371690 IP ip-10-0-2-140.eu-west-3.compute.internal.52600 > mynginx-5c7565866b-flfct.http: Flags [.], ack 854, win 481, options [nop,nop,TS val 1503500976 ecr 1488507084], length 0
13:52:55.372566 IP ip-10-0-2-140.eu-west-3.compute.internal.52600 > mynginx-5c7565866b-flfct.http: Flags [F.], seq 75, ack 854, win 481, options [nop,nop,TS val 1503500976 ecr 1488507084], length 0
13:52:55.372639 IP mynginx-5c7565866b-flfct.http > ip-10-0-2-140.eu-west-3.compute.internal.52600: Flags [F.], seq 854, ack 76, win 485, options [nop,nop,TS val 1488507085 ecr 1503500976], length 0
13:52:55.373227 IP ip-10-0-2-140.eu-west-3.compute.internal.52600 > mynginx-5c7565866b-flfct.http: Flags [.], ack 855, win 481, options [nop,nop,TS val 1503500978 ecr 1488507085], length 0
13:52:55.835262 IP6 fe80::98a8:75ff:fe4f:1767 > ff02::2: ICMP6, router solicitation, length 16
```
## References
- https://xxradar.medium.com/how-to-tcpdump-effectively-in-docker-2ed0a09b5406
- https://xxradar.medium.com/how-to-tcpdump-effectively-in-kubernetes-part-1-a1546b683d2f
- https://xxradar.medium.com/how-to-tcpdump-effectively-in-kubernetes-part-2-7e4127b42dc7
- https://xxradar.medium.com/tcpdump-nc-and-k8s-fun-1276414907b5
- https://xxradar.medium.com/how-to-tcpdump-using-ephemeral-containers-in-kubernetes-d066e6855785
- https://xxradar.medium.com/termshark-in-docker-d4cf15807b48
- https://xxradar.medium.com/mitmproxy-and-kubernetes-e897e903b1cb
