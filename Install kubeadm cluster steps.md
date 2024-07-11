# Install kubeadm cluster steps

## Base config

- configure prerequisites (*all nodes*)

  - disable swap ([Install Kubeadm doc page](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime))
  - setup ip forward and iptables for bridged traffic ([Container Runtime doc page](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#install-and-configure-prerequisites))

- install container runtime (*all nodes*)

  - containerd : install stuff and dependencies, specific systemd config ?
  - docker : 
    - install docker engine
    - install cri-dockerd (*note: use "advanced setup > install manually to avoid having to write a service for cri-dockerd"*)

- install kubeadm, kubelet and kubectl ([Install Kubeadm doc page](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)) (*all nodes*)

  - update repository and install prerequisites
  - add k8s repository and signing key
  - install and `apt-mark hold` les 3 packages

- run `kubeadm init` (*control plane only*)

  - *(optional) Pull images beforehand*  `sudo kubeadm config images pull`
  - for **docker**, specify unix domain socket (`kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock`)
  - for **high-availability** control plane : add `--control-plane-endpoint "<proxy-address>:<proxy-port>" --upload-certs`
  - if needed, specify `--pod-network-cidr` to avoid overlapping with the node cidr or to specify cidr requested by the chosen cni plugin (`kubeadm init --pod-network-cidr 192.168.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock` for docker)
  - for **TLS config** (useful for metrics server and stuff), use a config file with the kubeadm init command ([blog](https://particule.io/en/blog/kubeadm-metrics-server/#from-scratch))

- copy kube config (using the commands provided in the kubeadm init output)

- deploy CNI pod network addon (see [Installing Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/) for plugins list) (*control plane only*)

  - calico : `curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml -O` then `kubectl apply -f calico.yaml` (:warning: use 192.168/16 as cidr or edit yaml before applying at `CALICO_IPV4POOL_CIDR`)

- join other nodes

  - use `kubeadm join <cp-endpoint> --token <REDACTED> --discovery-token-ca-cert-hash sha256:<REDACTED> [--control-plane]`

  - to get the control plane endpoint, run `kubectl cluster-info`

  - to get the token value, run `kubeadm token list`

  - to get the hash, run ([doc](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes)) 

    ```bash
    openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
       openssl dgst -sha256 -hex | sed 's/^.* //'
    ```



## Optional config

- configure kubectl on other nodes (scp `.kube/config`)
- install metrics server (kubectl apply manifest, then edit deployment to add `--kubelet-insecure-tls`)
- to configure TLS after cluster was initialized without ssl config, see [here](https://particule.io/en/blog/kubeadm-metrics-server/#running-cluster)
