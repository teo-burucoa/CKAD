# Formation Kubernetes

WIP q44 https://dev.to/subodev/50-questions-for-ckad-and-cka-exam-3bjm
\-> timer : 1h21' + 2h21 (pas très focused sur la 2e partie)



## Révisions exam

### How 2 créer  des ressources rapidement

- deploy : `kubectl create deployment exp --image=busybox  [--port 80 --replicas 2 -- /bin/sh -c 'sleep 3600']` (:warning: quotes après commande sh)
- service : `kubectl expose mydeploy [--type NodePort] [--port 31000] [--target-port 80] [--name mysvc] [--dry-run=client -o yaml]`
- pod : `kubectl run mypod --image nginx [--labels "key1=val1,key2=val2"]`
  - pod + service :  `kubectl run web --image=nginx [--labels="app=web"] --expose --port=80`
  - probes : **doc** ("Configure Liveness, Readiness and Startup Probes")
  - volumes : **doc** ("Volumes", examples of config for each type)
- service account : `kubectl create sa mysa`
- (cluster)role : `kubectl create [cluster]role myrole --verb=get,list,watch --resource=pods,pods/status [--resource-name=readablepod]`
  - verbes pour rbac : ` kubectl api-resources -o wide [--sort-by name]`
- (cluster)rolebinding : `kubectl create [cluster]rolebinding my-binding --[cluster]role=myrole [--serviceaccount=my-namespace:my-sa] [--user=my-username]`
- configmap : `kubectl create cm my-cm [--from-file=path/to/bar --from-file=key1=/path/to/file1.txt --from-literal=key1=config1 --from-env-file=path/to/foo.env]`
  - usage with pod : **doc** ("Configure a Pod to Use a ConfigMap")
- daemonset : create a **deployment** yaml then edit to remove `spec.replicas` and `spec.strategy` + change kind to `DaemonSet`
  \-> or use the docs and trim down the example (*useful if need toleration to run on control plane*)
- ingress : ` kubectl create ingress NAME --rule=host/path[*]=service:port[,tls[=secret]]`
  ou **doc**
- limit ranges : **doc** "Limit Ranges" (~~kubectl create~~)
- persistent volume : **doc** ("Configure a Pod to Use a PersistentVolume for Storage")
- persistent volume claim : **doc** ("Configure a Pod to Use a PersistentVolume for Storage")
- storage class : **doc** ("Storage Classes")
- resource quota : `kubectl create quota my-quota --hard=cpu=1,memory=2G,pods=2`
- network policy : **doc** ("Network Policies")
- job : `kubectl create job my-job --image=busybox -- date`
- cronjob : `kubectl create cronjob my-job --image=busybox --schedule="*/2 * * * *" -- date`
- secret :
  - generic secret (from a file/directory/literal value) : `kubectl create secret generic my-secret [--from-file=path/to/bar] [--from-literal=key1=supersecret --from-literal=key2=topsecret]`
  - tls secret : `kubectl create secret tls tls-secret --cert=path/to/tls.cert --key=path/to/tls.key`
- security context : **doc**
- tolerations : **doc**
- statefulset : créer deploy dry puis changer `kind` et delete `spec.strategy`

### Install kubeadm cluster (k8s doc steps)

- configure prerequisites (*all nodes*)
  - disable swap ([Install Kubeadm doc page](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime))
  - setup ip forward and iptables for bridged traffic ([Container Runtime doc page](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#install-and-configure-prerequisites))
- install container runtime (*all nodes*)
  - containerd : install stuff and dependencies, specific systemd config ?
  - docker : 
    - install docker engine
    - install cri-dockerd
- install kubeadm, kubelet and kubectl ([Install Kubeadm doc page](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)) (*all nodes*)
  - update repository and install prerequisites
  - add k8s repository and signing key
  - install and `apt-mark hold` les 3 packages
- run `kubeadm init` (*control plane only*)
  - *(optional) Pull images beforehand*  `sudo kubeadm config images pull`
  - for **docker**, specify unix domain socket (`kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock`)
  - for **high-availability** control plane : add `--control-plane-endpoint "<proxy-address>:<proxy-port>" --upload-certs`
  - if needed, specify `--pod-network-cidr` to avoid overlapping with the node cidr or to specify cidr requested by the chosen cni plugin (`kubeadm init --pod-network-cidr 192.168.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock` for docker)
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

### Notes/tips à garder à l'esprit

- image nginx est basée sur Debian
  \-> possible d'utiliser apt pour installer des trucs

  ```sh
  apt update && apt install net-tools
  # puis pour voir sur quels ports le container écoute
  netstat -ntlp 	# note: marche aussi sur image busybox
  ```
- voir sur quel port un container écoute (inclus dans net-tools pour apt, et de base dans busybox) :

  ```sh
  netstat -ntlp
  ```
- alternative à busybox avec un package manager pour curl et autres : alpine

  ```sh
  kubectl create deploy alpine --image alpine -- sh -c 'sleep 3600'
  # dans le pod alpine
  apk --update add curl
  ```
- trouver différents fichiers de config kubelet : `systemctl status kubelet.service` puis suivre la ligne en-dessous de CGroup
  \-> donne la localisation du fichier de config kubelet, `/var/lib/kubelet/config.yaml` dans lequel on trouve notamment la localisation du fichier `resolv.conf` utilisé par coredns ou le `staticPodPath`
- snapshot save pour etcd : ~~dans le pod etcd~~ **depuis le control plane node** :

  ```sh
  sudo ETCDCTL_API=3 etcdctl snapshot save --endpoints <ip>:2379 snapshot.db --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key
  ```
- liste des ressources créables avec `kubectl create` :

  ```sh
  kubectl help create
  # puis exp de commande kucebtl create pour la ressource donnée
  kubectl help create cronjob
  ```
- voir les permissions d'un service account :

  ```sh
  k auth can-i get pods/anotherpod --as system:serviceaccount:default:pod-reader
  ```
- delete tous les pods failed (error,...) :

  ```sh
  k delete po --field-selector status.phase=Failed
  ```
- exposer des infos sur un pod à l'intérieur de ses containers :

  ```yaml
        env:
          - name: MY_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name: MY_POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
  ```
- le `sh -c 'mycommand'` avec `kubectl exec` sert à ce que `mycommand` soit interprétée par le container et pas le shell actuel (:warning: utiliser des single quotes)
  \-> a de l'importance quand :

  ```sh
  # exécuté par le shell actuel
  ubuntu@worker-1:~$ k exec -it tmp-867f56fc4c-nlnxf -- echo $CURR_HOST
  worker-1
  # exécuté par le container
  ubuntu@worker-1:~$ k exec -it tmp-867f56fc4c-nlnxf -- sh -c 'echo $CURR_HOST'
  tmp-container
  
  # autre exp
  ubuntu@worker-1:~$ k exec -it tmp-6cdd69b676-pcbql -- echo $(env)
  LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40; #[...]
  ubuntu@worker-1:~$ k exec -it tmp-6cdd69b676-pcbql -- sh -c 'echo $(env)'
  KUBERNETES_SERVICE_PORT=443 #[...]
  ```

### CNI plugin decision matrix

|          | Provider networking | Encapsulation and routing | Support for network policies | Datastore | Encryption | Ingress / Egress |
|----------|---------------------|---------------------------|------------------------------|-----------|------------|------------------|
| Flannel  | Layer 3             | VxLAN                     | No                           | ETCD      | Yes        | No               |
| Calico   | Layer 3             | BGP, eBPF                 | Yes                          | ETCD      | Yes        | Yes              |
| Weavenet | Layer 2             | VxLAN                     | Yes                          | NO        | Yes        | Yes              |
| Canal    | Layer 2             | VxLAN                     | Yes                          | ETCD      | No         | Yes              |

### Toolbox image cmd availability matrix

| Image           | run with sleep cmd ?      | shell bin | nslookup             | dig                         | netstat               | curl               |
|-----------------|----------------------------|-----------|----------------------|-----------------------------|-----------------------|--------------------|
| busybox         | :white_check_mark:         | sh        | :white_check_mark:   | :x:                         | :white_check_mark:    | :x:                |
| alpine          | :white_check_mark:         | sh        | :white_check_mark:   | `apk add --update bind-tools` | :white_check_mark:    | `apk add curl`       |
| nginx           | :x:                        | bash      | `apt install dnsutils` | `apt install dnsutils`        | `apt install net-tools` | :white_check_mark: |
| curlimages/curl | :white_check_mark:         | sh        | :white_check_mark:   | :x:                         | :white_check_mark:    | :white_check_mark: |

Note: if run with sleep cmd is true, use `-- sh -c "while true; do sleep 3600; done"` or some equivalent as command for the pod (required, otherwise will enter CrashLoopBackoff because the container will complete as soon as it starts)



### How 2 etcdctl backup/restore for real

- (facultatif) créer des ressources à backup sur le cluster
- récupérer l'endpoint de etcd avec le pod (alternative : voir dans `/etc/kubernetes/manifests/etcd.yaml`) :

  ```sh
  # find the name of the etcd pod
  kubectl -n kube-system get pods
  # then get the yaml of the pod to see the etcd command arguments
  kubectl -n kube-system get po <etcd-pod-name> -o yaml
  
  # alternative: get the pod directly using the label selector
  kubectl -n kube-system get po --selector component=etcd -o yaml
  
  # find the part with the command similar to:
      - command:
        - etcd
        - --advertise-client-urls=https://172.30.1.2:2379
        - --cert-file=/etc/kubernetes/pki/etcd/server.crt
        - ...
        - --data-dir=/var/lib/etcd
        - ...
        - --key-file=/etc/kubernetes/pki/etcd/server.key
        - --listen-client-urls=https://127.0.0.1:2379,https://172.30.1.2:2379
        - ...
        - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
  ```
- créer une snapshot de la db avec un endpoint récupéré dans `listen-client-urls` et les certificats/clés de la commande etcd :

  ```sh
  export ETCDCTL_API=3
  sudo etcdctl snapshot save --endpoint https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  backup.db
  ```
- delete des ressources dans le cluster k8s (pour voir la différence après restore)
- restaurer la snapshot etcd (ça crée une nouvelle instance etcd indépendante de l'existante), en précisant un nouveau data dir (default : `/var/lib/etcd` pour l'existant) :

  ```sh
  sudo etcdctl snapshot restore --data-dir /var/lib/restored-etcd backup.db
  ```
- update etcd du cluster k8s pour utiliser le data dir du nouveau etcd restauré :

  ```sh
  sudo vim /etc/kubernetes/manifests/etcd.yaml
  
  # ...
    - hostPath:
        path: /var/lib/etcd		# udpate avec le path du nouveau data-dir
        type: DirectoryOrCreate 
      name: etcd-data
  ```
- save & quit, attendre un peu avant que kube-apiserver soit de nouveau up (normal si kubectl timeout)
- (optionnel) si le pod `etcd-...` reste pending pendant un moment, le delete à la main (`kubectl delete pod ...`), il devrait revenir et être running

## Useful resources

- illustrated guide to k8s networking ([slideshow](https://speakerdeck.com/thockin/illustrated-guide-to-kubernetes-networking))
- K8s Networking webinar ([YT](https://youtu.be/GgCA2USI5iQ))
- killercoda training scenarios (good for prep) ([lien](https://killercoda.com/killer-shell-cka))
- network policy editor ([lien](https://editor.networkpolicy.io/))
- k8s training (official) [github](https://github.com/StenlyTU/K8s-training-official)
- json/yam path quizz ([link](https://kodekloud.com/courses/json-path-quiz/))
- CKA exam curriculum ([github](https://github.com/cncf/curriculum/blob/master/CKA_Curriculum_v1.27.pdf))
- exam
  - environment preview ([yt](https://youtu.be/9UqkWcdy140)) ([screenshots officiels](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2/exam-user-interface/examui-performance-based-exams))
  - allowed resources ([lien](https://docs.linuxfoundation.org/tc-docs/certification/certification-resources-allowed#certified-kubernetes-administrator-cka-and-certified-kubernetes-application-developer-ckad))
  - rules ([lien](https://docs.linuxfoundation.org/tc-docs/certification/lf-cert-agreement#4-2-exam-rules-and-policies)) (location rules, candidate conduct)
- cka exam study guide ([lien](https://devopscube.com/cka-exam-study-guide/))
- udemy course, utilisé par Tatel et niveau proche de l'exam (25€, [link](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/))
- sample questions ([link](https://k21academy.com/docker-kubernetes/cka-ckad-exam-questions-answers/))
- well-explained network policy examples ([github](https://github.com/ahmetb/kubernetes-network-policy-recipes))
- consignes détaillées et **bons tips** sur l'exam ([lien](https://github.com/fireflycons/tips-for-CKA-CKAD-CKS))
- 50 questions training + liens vers d'autres trainings ([lien](https://dev.to/subodev/50-questions-for-ckad-and-cka-exam-3bjm)) ([autres](https://github.com/subodh-dharma/ckad/tree/master))
- etcd backup & restore commands ([github](https://github.com/bmuschko/cka-crash-course/blob/master/exercises/04-etcd-backup-restore/solution/solution.md))
- **bons tips** sur des questions fréquentes du CKA ([github](https://github.com/kodekloudhub/community-faq/blob/main/docs/diagnose-crashed-apiserver.md))
- **exercices déployables** facilement sur troubleshooting kube-apiserver ([github](https://github.com/kodekloudhub/cka-debugging-api-server))

## Exam tips

- train on a local cluster with minikube/equivalent
- When you have to fix a component (like kubelet) in one cluster, just check how it's setup on another node in the same or even another cluster. You  can copy config files over etc
- for training : use imperative commands (`kubectl create`) over declarative (yaml) way (way faster)
- 6 min/question on average
- aliases & autocomplete :

  ```sh
  alias k=kubectl
  alias ks='k -n kube-system'
  alias krun="k run -h | grep '# ' -A2"
  alias kgp="k get pod" 
  alias kgd="k get deploy" 
  alias kgs="k get svc" 
  alias kgn="k get nodes" 
  alias kd="k describe" 
  alias kge="k get events --sort-by='.metadata.creationTimestamp' |tail -8"
  
  source <(kubectl completion bash)
  echo "source <(kubectl completion bash)" >> ~/.bashrc
  
  # terminal color on main terminal
  TERM=xterm-color
  
  export YAML="--dry-run=client -o yaml"
  # k create deploy test-deploy --image nginx $YAML > test-deploy.yaml
  ```
- read CKA curriculum beforehand :white_check_mark:
- watch exam environment preview :white_check_mark:
- learn to use resource shortnames

  > cm → configmaps
  > ds → daemonsets
  > deploy → deployments
  > ep → endpoints
  > ev → events
  > hpa → horizontalpodsautoscalers
  > ing → ingresses
  > limits  →limitranges
  > ns →namespaces
  > no →nodes
  > pvc → persistentvolumeclaims
  > pv →  persistentvolumes
  > po → pods
  > rs → replicasetsrcreplicationcontrollers  
  > quota → resourcequotas
  > sa → serviceaccounts
  > svc → services
- use dry run to general yaml (and use shell var for dry run yaml flag : `export do="--dry-run=client -o yaml"` -> `k run nginx --image=nginx $do > pod.yaml`)
- delete pods without waiting for graceful delete:

  ```sh
  export now="--force --grace-period 0"
  k delete pod test $now
  ```
- use `kubectl help <cmd>` for resource creation examples (`kubectl help run`),
- use `kubectl explain <resource>` to view the definition of a resource (or part of a resource)
  \-> ex: `k explain pod.spec.containers`
- env vars for ectdctl :

  ```sh
  export ECTDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
  export ECTDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
  export ECTDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
  export ETCDCTL_API=3
  ```
- arguments commande etcdctl pour avoir les ca cert et tout :

  ```sh
  kubectl -n kube-system get po etcd-controlplane -o=jsonpath='{.spec.containers[*].command}' |jq .
  ```
- possible d'ajouter des labels avec la commande impérative `run`

  ```sh
  kubectl run mypod --image nginx --labels "pod=cat,type=smol"
  ```
- utiliser `-o name` pour avoir rapidement les noms des ressources (marche sur tout ? notamment les contextes kubeconfig avec `kubectl config get-contexts -o name`)
- env vs envFrom avec secrets/configmaps :

  ```yaml
  envFrom:
  - secretRef:
      name: <secret name>
      
  env:
  - name: <variable name>
    valueFrom:
      secretKeyRef:
        name: <secret name>
        key: <Only this key and not all the keys in the secret>
  ```
- trouver les composants k8s managés par systemd :

  ```sh
  find /etc/systemd/system/ | grep kube
  # ou pour etcd
  find /etc/systemd/system/ | grep etcd
  ```
- exécuter une commande par ssh sur un hôte distant et récupérer son output sur la machine locale :

  ```sh
  ssh cluster1-node2 'crictl logs b01edbe6f89ed' &> /opt/course/17/pod-container.log
  ```
- curl en ignorant tls insecure : `-k`

  ```sh
  # depuis un pod du cluster (pour dns resolution)
  curl -k https://kubernetes.default.svc.cluster.local
  ```
- curl le kube-apiserver depuis laptop/machine externe :

  ```sh
  kubectl proxy --port=8080
  # second shell
  curl http://localhost:8080/api
  ```



### Useful aliases

- .bashrc

```sh
export dry="-o yaml --dry-run=client"
export now="--force --grace-period 0"

alias kgp="kubectl get pods"
alias kn='kubectl config set-context --current --namespace '
alias ks="kubectl -n kube-systems"

source <(kubeadm completion bash)
```

setup .tmux.conf :

```sh
cat << EOF > ~/.tmux.conf
set -g mouse on
bind -n C-s setw synchronize-panes
EOF
```

setup .vimrc :

```
set expandtab
set ts=2
set sw=2
set pastetoggle=<F3>
```



## Webinar: Kubernetes and Networks: Why is it so dang hard ?

network models (more exist tho) :

**fully-integrated / "flat" mode** : each node has its own IP address and gets an IP range
\-> everything (pods, nodes,...) share the same IP space, same network, no translation, no overlay network
=> easier for communications
\-> **good** when :

- ip space readily available (cluster usually needs big ranges like /16) -> if IPv6 usable, usually best choice !
- network programmable/dynamic
- high integration/performance is needed
- K8s is a large part of the footprint/budget

\-> **not good** when:

- ip fragmentation/scarcity (hard/impossible to find /16)
- hard to configure network infra
- K8s = small part of the footprint

**fully-isolated ("air-gapped")** : cluster networks are completely isolated, no connectivity from inside to outside at all
\-> can reuse all IP addresses !
\-> good when :

- workloads need isolation, no need for integration
- IP space scarce/fragmented
- network not programmable/dynamic
- need easy to setup/strong security boundaries

\-> bad when :

- need communication across a cluster-edge

**bridged-mode ("island mode")** : each cluster has their private network and pods within get IPs from this IP range, BUT there are gateways that link the clusters to the outside
\-> very common model ; can be implemented via overlays (VXLAN,...) or can use private routing rules (private BGP advertising, static routing,...)
\-> good when :

- IP space scarce/fragmented (can afford node IP addresses but not pod IPs)
- need some integration
- network not programmable/dynamic

\-> bad when :

- need to debug connectivity (because of translation in gateways)
- need direct-to-endpoint communications (e.g. kafka, systems that assume no load balancers,...)
- need a lot of services exposed (especially non-HTTP) : requires a lot of infra
- rely in client IPs for firewalls
- large number of nodes (overlay/route advertising scale poorly ?)

various forms of **gateway modes** that can be used with island mode :

- nodes : no extra load balancer/router ; nodes have 1 leg in the main network (-> node IP) and 1 in the pod network
- ingress service nodeports : all nodes listen on the same ports for a given service -> nodes use IP `dst_port` to route traffic to the correct service (DNAT = destination network address translation) ; then rewrites the IP packet (iptables/ipvs/nf tables/ebpf) and forwards it to the dest pod -> common pattern : ingress all traffic into an L7 proxy (envoy/nginx) on a nodePort inside the cluster that then routes the traffic inside the cluster (e.g. in-cluster ingress controller)
- egress IP masquerade (aka SNAT, source NAT) : traffic leaving the node looks like it came from this machine (not from a specific pod inside), obscures client IP => ingress nodeport + egress IP masquerade costs 2 translations
- VIP (ingress) : used in some cloud providers, gives a virtual IP instead of needing to know node + port ; VIP represents a service in k8s -> similar to nodeport, but node uses IP `dst_ip` to route instead of `dst_port` -> more compatible with traditional DNS -> still needs something like SNAT for egress
- proxy (ingress) : VIP typically does not terminate TCP sessions (will forward packets), while proxy *does terminate* the TCP session and open a new session -> can either route to NodePort (ex: AWS ELB) or directly to pod IPs -> pb: proxy obscures client IP (traffic appears to come from the proxy's IP), need an additional mechanism to retain this info -> AWS ELBs can encode client IP in HTTP headers -> still need something like SNAT for egress

**archipelago mode** ("big island") : within the archipelago, the model is the flat model : flat space, multiple clusters can talk to each other without translation ; but becomes island mode when integrated with the rest of the network
\-> like island mode, can be implemented as overlay or not, **still need gateways** (see above for solutions, similar gateway options)
\-> can't reuse pod IPs between clusters but can between archipelagos
\-> good when :

- need high integration across clusters
- need some integration with non-k8s
- IP space scarce/fragmented (but uses more space than plain island mode)
- network not programmable/dynamic

\-> bad when:

- need to debug connectivity
- need direct-to-endpoint connectivity outside the cluster
- need to expose a lot of services to non-k8s env (ex: nodePorts limited to 2k services by default)
- rely on client IPs for firewalls
- large number of nodes across all clusters (scaling limit = nb of nodes in the archipelago // island mode : nb of nodes in the island)

## Kubernetes Fundamentals (Linux Foundation)

progress: ch3 installing with kubeadm

### Ch3 : Installation & config

using kubeadm : https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

containerd install steps : https://github.com/containerd/containerd/blob/main/docs/getting-started.md

générer config.toml containerd : `containerd config default | tee /etc/containerd/config.toml >/dev/null 2>&1` (ou `containerd config default > /etc/containerd/config.toml` en root ?)

disable swap :

```sh
sudo swapoff -a
# replace fstab entry for persistent config
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

init cluster avec kubeadm : `kubeadm init --pod-network-cidr=10.1.1.0/24 --apiserver-advertise-address <IP address of the Master node> [--config kubeadm-config.yaml]` (config : affaire de configurer le cgroup driver de kubelet, par ex pour systemd pour 1.21-)

- installation containerd facile sur ubuntu :

  ```sh
  apt-get update &&  apt-get install containerd.io -y
  containerd config default | tee /etc/containerd/config.toml
  sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
  systemctl restart containerd
  ```
- générer le hash du discovery token ca cert et utiliser pour kubeadm join :

  ```sh
  # optionnel, si token expiré
  sudo kubeadm token create
  
  sudo openssl x509 -pubkey \
  -in /etc/kubernetes/pki/ca.crt | openssl rsa \
  -pubin -outform der 2>/dev/null | openssl dgst \
  -sha256 -hex | sed 's/ˆ.* //'
  
  # puis utiliser pour join
  kubeadm join \
  --token REDACTED \
  hostname:6443 \
  --discovery-token-ca-cert-hash \
  sha256:REDACTED
  ```
- lister endpoints :  `kubectl get ep`
  \-> (ex d'un service) : ClusterIP est donné par le CNI, alors que l'endpoint est donné par kubelet et kube-proxy (adresses différentes !)

### Ch4 : Kubernetes Architecture

"When building a cluster using **kubeadm**, the **kubelet** process is managed by **systemd**. Once running, it will start every pod found in `/etc/kubernetes/manifests/`."

control plane :

- **kube-apiserver** is the **only** connection to the etcd database -> validates & configures data for api objects, services REST operations -> "control plane process for the entire cluster", and frontend of the cluster's shared state
- **kube-scheduler** : determines which node will host a pod
- **etcd** database : stores the state of the cluster, networking and other persistent info (uses b+tree key-value store) -> works with curl
- kube-controller-manager  : core control loop daemon that interacts with kube-apiserver to determine the state of the cluster (and contacts the necessary controller if the state does not match)
- cloud-controller-manager

worker nodes :

- kubelet : interacts with the container engine on the node, makes sure containers that need to run are running (receives pod specs, dowloads/mounts/gives access to storage/secrets/configmaps and sends back status to the kube-apiserver)
- kube-proxy : manages the network connectivity to the containers using iptables entries -> when the network config is changed (ex: a new pod has been created), the kube-proxy receives new info from the kube-apiserver (all kube-proxy components everywhere in the cluster get the update)

additional optional stuff :

- fluentd : unified cluster-wide logging layer
- supervisord : for non-systemd clusters, daemon that monitors kubelet and docker processes

**operators** : aka controllers/watch-loops ; ~1 per k8s resource type ?

> Kubernetes' [operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) concept lets you extend the cluster's behaviour without modifying the code of Kubernetes itself by linking [controllers](https://kubernetes.io/docs/concepts/architecture/controller/) to one or more custom resources. Operators are clients of the Kubernetes API that act as controllers for a [Custom Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

\-> ex: *service* operator listens to the *endpoint* operator to provide a persistent ip for pods

`ResourceQuota` : object that can be created to set soft & hard resource usage limits in a namespace (manages mores than just memory and CPU, and limits several objects)

calico stuff :

- Felix : primary calico agent on each machine
- BIRD : dynamic IP routing daemon used by Felix to read routing state and distribute that information to other nodes in the cluster

control plane node must run linux, but worker nodes can be windows server 2019

`Node` objects namespace : kube-node-lease

pause container : container that holds the network namespace for its pod (reserves its IP address in the namespace)
\-> holds the ephemeral pod IP (-> endpoint = pod IP + port, created together with a service)

Gros niaisage de etcdctl pour le backup :

- utiliser `export ETCDCTL_API=3` (ou d'autres exports) uniquement si déjà root/pas besoin de sudo
  \-> utiliser sudo ignore les variables env exportées !!
- solution non-root depuis control plane host (:warning: `sudo` AVANT de set la variable !!!)
  (remplacer endpoint par l'ip du node control plane obtenue avec `k get no -o wide`

  ```sh
  sudo ETCDCTL_API=3 etcdctl snapshot save etcd_backup.db \
    --endpoints=https://10.0.0.6:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key
  ```
- solution exec dans container etcd (namespace `kube-system`) :

  ```sh
  kubectl -n kube-system exec -it <container-name> -- sh \
  -c "ETCDCTL_API=3 \
  ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt \
  ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt \
  ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key \
  etcdctl --endpoints=https://127.0.0.1:2379 snapshot save /var/lib/etcd/snapshot.db "
  ```

  (note: `/var/lib/etcd/` et `/etc/kubernetes/pki/etcd/ `sont sur des volumes HostPath donc accessibles depuis la machine hôte aussi)

generate artificial ressource stress : use `vish/stress` image
\-> exp config:

```yaml
  template:
    metadata:
      labels:
        app: stress
    spec:
      containers:
      - image: vish/stress
        name: stress
        resources:
          limits:
            cpu: "1"
            memory: "4Gi"
          requests:
            cpu: "0.5"
            memory: "2500Mi"
        args:
          - -cpus
          - "2"
          - -mem-total
          - "950Mi"
          - -mem-alloc-size
          - "100Mi"
          - -mem-alloc-sleep
          - "1s"
```

### Ch5: APIs and access

- curl the API(s) :

  ```sh
  # root
  sudo curl https://k8s-control:6443 \
  --cert /etc/kubernetes/pki/apiserver-kubelet-client.crt \
  --key /etc/kubernetes/pki/apiserver-kubelet-client.key \
  --cacert /etc/kubernetes/pki/ca.crt
  
  # some path found in the root response, then in subsequent calls
  sudo curl https://k8s-control:6443/apis/cilium.io/v2 --cert /etc/kubernetes/pki/apiserver-kubelet-client.crt --key /etc/kubernetes/pki/apiserver-kubelet-client.key --cacert /etc/kubernetes/pki/ca.crt
  ```
- checking access : `k auth can-i create <resource-type> [--as bob] [-n somenamespace]`

infos de .kube/config :

- `clusters` : nom et endpoint de chaque cluster répertorié (+ `certificate-authority-data`)
- `contexts` : associations entre un user et un cluster (tous les 2 répertoriés dans la config) et éventuellement un namespace (?) pour switch entre plusieurs clusters avec des profils différents
- `current-context` : contexte utilisé par kubectl

special namespaces :

- `kube-node-lease` : where worker node lease information is kept
- `kube-public` : namespace readable by all, even unauthenticated clients -> general info
- `kube-system` : infrastructure pods

#### Configuring TLS access

- create variables using cert information

  ```sh
  export client=$(grep client-cert $HOME/.kube/config |cut -d" " -f 6)
  export key=$(grep client-key-data $HOME/.kube/config |cut -d " " -f 6)
  export auth=$(grep certificate-authority-data $HOME/.kube/config |cut -d " " -f 6)
  
  # alternative à cut:
  export client=$(grep client-cert $HOME/.kube/config |awk '{print $2}')
  ```
- encode keys for use with `curl`

  ```sh
  echo $client | base64 -d - > ./client.pem
  echo $key | base64 -d - > ./client-key.pem
  echo $auth | base64 -d - > ./ca.pem
  ```
- get API server URL

  ```bash
  kubectl config view |grep server
  ```
- use curl with encoded keys to connect to the API server (replace IP)

  ```sh
  curl --cert client.pem --key client-key.pem --cacert ca.pem \
  	https://10.0.0.6:6443/api/v1/pods
  ```
- create a pod using a json spec

  ```sh
  curl --cert client.pem --key client-key.pem --cacert ca.pem https://10.0.0.6:6443/api/v1/namespaces/default/pods -XPOST -H'Content-Type: application/json' -d@curlpod.json
  ```

  <details><summary>JSON spec</summary>
  json
  {
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
  "name": "curlpod",
  "namespace": "default",
  "labels": {
  "name": "examplepod"
  }
  },
  "spec": {
  "containers": [
  {
  "name": "nginx",
  "image": "nginx",
  "ports": [
  {
  "containerPort": 80
  }
  ]
  }
  ]
  }
  }
  </details>

### Ch6: API objects

- resource quota vs limit range:

  > The **resource quota** is the **total** allocated resources for a **particular  namespace**, while **limit range** also used to assign **limits for objects like containers** (Pods) running inside the namespace.
- endpoint : represents the set of IPs for pods that match a particular service
- deployment vs statefulset :

  > How this is different than a Deployment is that a StatefulSet considers  each Pod as unique and provides ordering to Pod deployment.

  \-> statefulsets : default deployment scheme is sequential, pods are not deployed in parallel
- horizontal pod autoscaler : scale replication controllers, replicasets or deployments based on a target of 50% CPU usage by default (usage checked every 30 secs by kubelet and retrieved by the metrics sercer API call every minute)
  \-> "The autoscaler does not collect the metrics, it only makes a request for the aggregated information and increases or decreases the number of  replicas to match the configuration. "
- ClusterAutoscaler : adds/removes nodes to the cluster based on the inability to deploy a pod or having nodes with low utilization for at least 10 minutes
  \-> allows dynamic requests of resources from the cloud provider
- Jobs : run a set number of pods to completion (restart a pod if failed and number of completion not reached)
- CronJobs : use the same syntax as Linux cronjobs

RESTful API access :

- get IP & port of a node running a repica of the API server : `k config view |grep server`
- create a token : `export token=$(kubectl create token default)`
- test fetching basic API info : `curl https://10.0.0.6:6443/apis --header "Authorization: Bearer $token" -k` (note: `-k` pour insecure, pas vérifier les certificats TLS) -> cannot get info for endpoints such as `/api/v1/namespaces` (uses the `system:serviceaccount:default:default` user)
- pods can use certificates (included by k8s) to use the API -> certificates mouted at `/var/run/secrets/kubernetes.io/serviceaccount/` in pods

#### Using the Proxy

proxy between localhost and the k8s api server
\-> can be run from a node or from within a pod using a sidecar

- start the proxy to proxy all of the k8s api (and nothing else) : `k proxy --api-prefix=/ &`
- check connectivity to the kube api server through the proxy : `curl http://127.0.0.1:8001/api/`
- curl an endpoint that was not accessible previous with the default token : `curl http://127.0.0.1:8001/api/v1/namespaces` -> now works because the proxy makes the request on our behalf
- kill the proxy `sudo kill <PID>` (get PID from `sudo lsof -Pni |grep 8001` or `grep kubectl`)

basic job spec :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  completions: 5		# optional
  parallelism: 2		# optional
  activeDeadlineSeconds: 15		# optional, kills all pods if part of the job still running after deadline
  template:
    spec:
      containers:
        - name: resting
          image: busybox
          command: ["/bin/sleep"]
          args: ["3"]
      restartPolicy: Never
```

### Ch7: Managing state with deployments

default controller for `kubectl run`: deployment

ReplicationControllers ensure that a specified number of pod replicas is running at any one time
\-> updates managed on client-side -> issue if client loses connectivity
=> server-side updates of pods : deployments, which generate replicasets (more selection features than replication controllers, such as `matchExpressions`)

controllers run as a thread of the kube-controller-manager

edit to a deployment : new replicaSet is created (deploying the pods with the new podSpec), and the deployment will direct the old replicaset to shut down pods as the new replicaset pods become available (then terminates the old replicaset once done)

annotations **cannot** be used by kubectl to select an object (unlike labels that can)

deployment config :

- `metadata`: 
  - `generation` : records how many times the object has been edited
  - `resourceVersion` : used by etcd to manage object concurrency (value changes at every db change)
- `spec` : 
  - `progressDeadlineSeconds` : time until a progress error is reported during a change (could be because of quotas, image issues, or limit ranges)
  - `replicas` : determines how many pods are created by the managed replicaset
  - `revisionHistoryLimit` : how many old replicaset specs to retain for rollback
  - `selector` : collection of values ANDed together (all must be satisfied for the replica to match) -> :warning: do not create pods which match the selectors or the deployment will try to control these resources
  - `matchLabels` : set-based requirement of the pod selector (usually found with `matchExpressions`)
  - `strategy` : header for values having to do with updating pods 
    - `rollingUpdate` controls how many pods are deleted at a time for ex) 
      - `maxSurge` : max number of pods over desired number of pods (default 25%) to manage number of new created pods before deleting old ones
      - `maxUnavailable` : nb or % of pods which can be in a state other than Ready during the update process
    - `type` : type of the object configured (ex: RollingUpdate)
  - `template`: data passed to the replicaset to determine how to deploy an object (metadata and stuff) -> `spec` part of the `template` 
    - `containers`: 
      - `imagePullPolicy` : policy passed to the container engine about when/if an image should be downloaded or used from the local cache
      - `name` : leading stub of the Pod names (a unique string will be appended)
      - `resources` : where resource restrictions/requests/settings go
      - `terminationMessagePath`: location of where to output success/failure info for the container
      - `terminationMessagePolicy` : defaults to `File`, can be set to `FallbackToLogsOnError` (will use the last chunk of container log if the message file is empty and the container shows an error)
    - `dnsPolicy` : determines if dns queries should go to `coredns` or if `Default` will use the node's DNS resolution config
    - `scheduleName` : allows for the use of a custom scheduler instead of the K8s default
    - `securityContext` : allows to pass multiple security settings (SELinux context, AppArmor values, user UIDs for the containers to use)
    - `terminationGracePeriodSeconds` : amount of time to wait for a `SIGTERM` to run until a `SIGKILL` is used to terminate the container

deployment config status :

- `availableReplicas` : how many replicas were configured by the replicaset (to be compared with `readyReplicas` for actually ready replicas)
- `observedGeneration` : shows how often the deployment has been updated -> to understand the rollout and rollback situation of the deployment

rollbacks :

- use `kubectl annotate deploy mydeploy kubernetes.io/change-cause="kubectl create deploy mydeploy --image=nginx"`
- set the image of a deployment : `kubectl set image deploy mydeploy container1=image1 [container2=image2]`
- undo rollout : `kubectl rollout undo deploy mydeploy`
- rollout status :  `kubectl rollout status deploy mydeploy ` -> can also use `pause` and `resume` instead of `status`

#### Working with ReplicaSets

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-one
spec:
  replicas: 2
  selector:
    matchLabels:
      system: ReplicaOne
  template:
    metadata:
      labels:
        system: ReplicaOne
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.1
        ports:
        - containerPort: 80
```

delete the replicaset without deleting its child pods : `k delete rs rs-one --cascade=orphan`

rollingUpdate strategy :

```yaml
updateStrategy:
  rollingUpdate:
    maxUnavailable: 1
  type: OnDelete
```

:information_source: useful : permettre à des pods d'être scheduled sur les nodes control plane :

```yaml
spec:		# pod spec
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: "Exists"
      effect: "NoSchedule"
```

### Ch8 : Volumes and Data

a *volume* shares its pod's lifetime (not *containers*) // *persistent volume* : distinct lifetime that outlives the pod's

:warning: *no concurrency checking* for volumes

volume access modes (specified in PersistentVolume definition) :

- ReadWriteOnce: allows read-write by a single node
- ReadOnlyMany: allows read-only by multiple nodes
- ReadWriteMany: allows read-write by many nodes.

a few existing volume types:

- `emptyDir` / `hostPath` : easy to use
- NFS, iSCI : straightforward choices for multiple readers scenarios
- cloud provider stuff, block storage solutions,...

PVs are not namespaced but PVCs are

#### Secrets

:warning:A secret is not encrypted, only base64-encoded, by default.
\-> You must create an **EncryptionConfiguration** with a key and proper identity. + need to set `--encryption-provider-config` flag for the kube-apiserver

\-> use a secret as environment variable:

```yaml
...
spec:
  containers:
  - image: ...
    name: ...
    env:
    - name: MYQSL_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql
          key: password
```

\-> or use secrets as mounted files :

```yaml
spec:
  containers:
  - image: ...
    name: ...
    volumeMounts:
    - mountPath: /mysqlpassword
      name: mysql
  volumes:
  - name: mysql
    secret:
      secretName: mysql
```

\-> secrets are stored in the `tmpfs` storage on the host node (only if it is running a pod that needs it)

#### ConfigMaps

can also be used as env vars or mounts :

- as environment variables (use `printenv` inside the container to check values) :

  ```yaml
  env:
  - name: SPECIAL_LEVEL_KEY
    valueFrom:
      configMapKeyRef:
        name: special-config
        key: special.how
  # or
  envFrom:
  - configMapRef:
      name: special-config
  ```
- as volumes/mounts:

  ```yaml
  volumes:
  - name: config-volume
    configMap:
      name: special-config
  ```

create a configmap using the CLI :

```sh
k create configmap colors \
 --from-literal=text=black \
 --from-file=./favorite \
 --from-file=./primary/
```

\-> ex of result structure :

```sh
ubuntu@k8s-control1:~$ ll |grep 'best\|myfolder'
-rw-rw-r--  1 ubuntu ubuntu         7 Jun 19 18:00 best
drwxrwxr-x  3 ubuntu ubuntu      4096 Jun 19 18:05 myfolder/

ubuntu@k8s-control1:~$ tree
.
├── best
├── calico.yaml
├── myfolder
│   ├── file1
│   ├── file2
│   └── mysubfolder
│       └── file3
├── pod.yaml
├── role.yaml
└── vars.env


k create cm mycm --from-file=best,myfolder --from-literal mykey=myvalue
```

```yaml
data:
  best: |	# multiline content even though only 1 line in file
    gandhi
  file1: |
    filecontent1
  file2: |
    filecontent2
  # note: subfolders do not appear
  mykey: myvalue # single-line content
```

\-> folder gets treated as a list of files that behave as if they were input as `--from-file myfile` each
:warning: folder hierarchy not supported (subfolders not included)

### Ch9 : Services

quickly create a service for a deployment: `kubectl expose deployment nginx --port=80 --type=NodePort`

ExternalName service : has no selectors and does not define ports/endpoints
\-> allows the return of an alias to an external service (DNS-level redirection, no proxy/forward)
\-> can be useful for services not yet brought into the k8s cluster

:information_source:The **kubectl proxy** command creates a local service to access a ClusterIP. This can be useful for troubleshooting or development work.

Endpoint operator: queries for the ephemeral IP addresses of pods with a particular label

- The **ClusterIP** service configures a **persistent IP** address and directs traffic sent to that address to the existing pod's ephemeral addresses.  This only handles **inside the cluster** traffic.
- When a request for a **NodePort** is made, the operator **first creates a  ClusterIP**. After the ClusterIP has been created, a **high numbered port** is determined and a firewall rule is sent out so that traffic to the high  numbered port on **any node** will be sent to the persistent IP, which then  will be sent to the pod(s).
- A **LoadBalancer** does not create a load balancer. Instead, it creates a  **NodePort** and makes an **async request** to use a load balancer. If a  listener sees the request, as found when using public cloud providers,  one would be created. Otherwise, the status will remain *Pending* as no load balancer has responded to the API call.
- **ingress controller** : microservice running in a pod, **listening to a  high port** on whichever node the pod may be running, which will send traffic to a Service based on the URL requested

rewrite DNS rules to redirect `servicename.test.io` to `servicename.default.svc.cluster.local` :

```sh
kubectl -n kube-system edit configmap coredns

# >>> coredns configmap
.:53 {
	# add the line below
    rewrite name regex (.*)\.test\.io {1}.default.svc.cluster.local 

	# or 
	rewrite stop {
          name regex (.*)\.test\.io {1}.default.svc.cluster.local
          answer name (.*)\.default\.svc\.cluster\.local {1}.test.io
    }
```

### Ch11 : Ingress

instead of individual services for each of the pods, setup a controller that handles all of the traffic
\-> 2 (+?) supported : nginx and GCE stuff

efficiency of the ingress controller : instead of using lots of services, route traffic based on request host or path -> centralization of many services to a single point

many ingress controllers : gke, nginx, traefik, envoy,... -> anything that can reverse proxy

typical ingress :

```yaml
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress 
  metadata:
    name: ghost
spec:
  ingressClassName: nginx
  rules:
  - host: ghost.192.168.99.100.nip.io
    http:
      paths:
      - backend:
          service:
            name: ghost
            port:
              number: 2368 
        path: /
        pathType: ImplementationSpecific
```

> For more complex connections or resources such as service discovery,  rate limiting, traffic management and advanced metrics, you may want to  implement a service mesh.
>
> A *service mesh* consists of edge and embedded proxies  communicating with each other and handling traffic based on rules from a control plane. Various options are available, including Envoy, Istio,  and linkerd.

linkerd install :

```sh
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
# next steps are shown in the previous command's output
export PATH=$PATH:/home/ubuntu/.linkerd2/bin
linkerd check --pre 
linkerd install --crds | kubectl apply -f - 
linkerd install | kubectl apply -f -
linkerd check
linkerd viz install | kubectl apply -f -
linkerd viz check
linkerd viz dashboard &

# make web dashboard accessible from outside localhost
kubectl -n linkerd-viz edit deploy web
# puis delete tout après le "=" dans '- -enforced-host=[...]'

# edit service settings
kubectl edit svc web -n linkerd-viz
# puis edit port "http" :
#   - name: http
#     nodePort: 31500 		< ajouter (valeur au choix)
#     port: 8084
#     protocol: TCP
#     targetPort: 8084
#
# et edit type en NodePort au lieu de ClusterIP
```

faire en sorte que linkerd observe un objet :

```sh
k get deployments.apps nginx -o yaml |linkerd inject - |k apply -f -
```

\-> lab ingress :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-test
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: www.external.com
    http:
      paths:
      - backend:
          service:
            name: my-nginx-deployment
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
```

\-> send traffic to the ingress controller :

```sh
# get the IP of an ingress controller pod 
k get po -A -o wide |grep ingress
ingress-nginx   ingress-nginx-controller-6hkcv            1/1     Running   0                3h59m   10.0.1.135   k8s-worker    <none>           <none>
# or check the ingress controller's IP
k get ep -n ingress-nginx
NAME                                 ENDPOINTS                      AGE
ingress-nginx-controller             10.0.1.135:443,10.0.1.135:80   4h19m
ingress-nginx-controller-admission   10.0.1.135:8443                4h19m

# curl that IP without headers : 404
curl 10.0.1.135
<html>
<head><title>404 Not Found</title></head>

# curl the same IP with the Host header
curl 10.0.1.135 -H "Host: www.external.com"
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

### Ch12 : Scheduling

:warning: predicates & priorities are **deprecated** (available in 1.23- only)

~~scheduler uses filters (predicates) to find available nodes, then ranks each node using priority functions -> the node with the highest rank is selected to run the pod~~
\-> priority functions have been replaced by plugins (ImageLocality, NodeName, PodTopologySpread,...)

> order in which predicates are evaluated is **configurable** (to exclude nodes from unnecessary checks if not in the right condition)
> \-> use config of `kind: Policy` to order predicates or give special weights to priorities
>
> `hardPodAffinitySymmetricWeight` : if set pod A to run with pod B, then pod B should automatically be run with pod A
>
> priorities : functions that weight resources

\-> utiliser kube-scheduler (cli) sur un cluster kubeadm :
`kubectl -n kube-system exec -it kube-scheduler-<node-name> -- kube-scheduler -h`

pod specs that inform scheduling :

- `nodeName` and `nodeSelector` : assigns the pod to a specific node / a group of nodes with particular labels (all selectors must be met)
- `affinity` (see `k explain pod.spec.affinity`) : define node affinity of pod affinity/anti-affinity -> either with prefered or required affinity settings (soft / hard requirements)
- `taints` and `tolerations` : taints marks nodes so that pods cannot be scheduled on them except if they have the corresponding toleration
- `schedulerName` : allows to choose a custom scheduler profile

extension point : 12 stages of scheduling -> plugins can be used to modify how that state of the scheduler works

podAffinity example :

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
```

\-> The Pod can be scheduled on a node **running a Pod with a key label of security** and a value of **S1**. If this requirement is not met, the Pod will remain in a **Pending** state.

podAntiAffinity example :

```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100		# can range from 1 to 100
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: security
          operator: In
          values:
          - S2
```

nodeAffinity example :

```yaml
spec:
  affinity:
    nodeAffinity: 
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: diskspeed
            operator: In
            values:
            - quick
            - fast 
```

A node with a particular **taint** will **repel Pods without tolerations for that taint**. A taint is expressed as **key=value:effect**. The key and the value are created by the administrator.
\-> 3 possible effects :

- NoSchedule : if any, existing pods continue to run
- PreferNoSchedule : scheduler avoids this node unlais no untained nodes for the pods toleration ; does not affect existing pods
- NoExecute : existing pods will be evacuated and no future pods scheduled (unless toleration)

:information_source:

- If an empty key uses the **Exists** operator, it will tolerate every taint
- If there is no effect, but a key and operator are declared, all effects are matched with the declared key.

example of a time-limited toleration :

```yaml
tolerations:
- key: "server"
  operator: "Equal"
  value: "ap-east"
  effect: "NoExecute"
  tolerationSeconds: 3600
```

\-> the pod will remain on the server with a key of `server` and value `ap-east` for 3600 seconds after the node has been tainted with `NoExecute` (then gets evicted)

at the end of the scheduling process, the pod gets a binding that specifies which node it should run on
\-> technically could still schedule a pod without any scheduler by specifying a binding for that pod

count the number of pods running on a node (containerd) : `sudo ctr -n k8s.io c ls |wc -l`

### Ch13 : Logging and Troubleshooting

- run an ephemeral (debug) container inside a coredns pod *using process namespace sharing* (`ps` shows processes for the other container too)

  ```sh
  k -n kube-system debug -it --image=busybox coredns-5d78c9869d-4b4km --target=coredns # target: name of the targeted container
  
  / # ps aux
  PID   USER     TIME  COMMAND
      1 root      0:03 /coredns -conf /etc/coredns/Corefile
     14 root      0:00 sh
     20 root      0:00 ps aux
  
  
  # without process namespace sharing
  k -n kube-system debug -it --image=busybox coredns-5d78c9869d-4b4km
  
  /etc # ps aux
  PID   USER     TIME  COMMAND
      1 root      0:00 sh
     12 root      0:00 ps aux
  ```

kubectl's plugin manager : `krew`

find logs for k8s containers :

```sh
sudo find / -name "*coredns*log"   # or "*coredns*log"
# use less to read the returned file(s)
sudo less $(sudo find / -name "*proxy*log")
# or list all log files
sudo ls /var/log/containers/
sudo ls /var/log/pods/
```

\-> if  not on a k8s cluster that used systemd, logs will not be collected via journalctl ad can instead be found at `/var/log/kube-apiserver.log` (or kube-scheduler/kube-controller-manager)

find worker node files on non-systemd systems :

```
/var/log/kubelet.log
/var/log/kube-proxy.log
```

### Ch14 : Custom Resource Definitions

2 ways of adding custom resources to a k8s cluster :

- adding a custom resource definition (CRD) -> easier but less flexible
- aggregated APIs (AA) -> more tedious, requires a new API server to be written and added to the clsuter

:information_source: If you are using RBAC for authorization, you probably will need to grant access to the new CRD resource and controller

config example :

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.stable.linux.com	# syntax : <plural-name>.<group>
spec:
  group: stable.linux.com	# REST API: /apis/<group>/<version>
  version: v1
  scope: Namespaced
  names:
    plural: backups
    singular: backup
    shortNames:
    - bks
    kind: BackUp
```

new object config example :

```yaml
apiVersion: "stable.linux.com/v1"
kind: BackUp
metadata:
  name: a-backup-object
spec:		# depend on the controller for this kind
  timeSpec: "* * * * */5"
  image: linux-backup-image
replicas: 5
```

\-> the object's controller checks the specs:

- if validation is configured by the controller, the syntax of provided arguments is checked (ex: "* * * * */5" above) and an error will be thrown if the syntax does not match the expected expression
- if validation is not configured, only the existence of the variable/argument (?) will be checked (not its details/value)

asynchronous pre-delete hooks : `Finalizer` -> tells the controller to clean up resources before deleting the object itself (remains in terminating state during the cleanup)
\-> can be used for garbage collection, or to prevent deletion of resources

ex of finalizer : `kubernetes.io/pv-protection` that prevents accidental deletion of PersistentVolumes -> when a PV is used by a pod, k8s adds this finalizer to the pc

ex of crd for crontabs :

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            cronSpec:
              type: string
            image:
              type: string
            replicas:
              type: integer
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

describe custom resource definition : `kubectl describe crd <my-custom-resource>`

### Ch15 : Security

3 steps of an API call to the kube-apiserver:

- authentication : with an x509 certificate, a token or a webhook to an LDAP server
- authorization : handled by RBAC, or by webhook the two main authorization modes
- admission control : look at/modify the incoming request (+ final accept/deny)

authentication step : users are not created by the API and should be managed by an external system

admission control : see activated plugins with ` sudo grep admission /etc/kubernetes/manifests/kube-apiserver.yaml`

cert/key files :

- `--client-ca-file` in the kube-apiserver config : certificate authority/ies for validating client certificates
  \-> the CA that issues client certs (that are used for k8s api server authentication)
  \-> *not the same as the TLS certs' CA*
- `--tls-cert-file` and `--tls-private-key-file` : certificate (amd matching private key) files for TLS connection

  \-> the CA that issues the API server's TLS cert must be specified in the kubeconfig file

certificats X509 existent en 2 types :

- end-entity certificate : lie une identité à une clé publique (mais ne peut pas créer d'autres certificats)
- CA certificate : peut délivrer d'autres certificats

**security contexts** can be used to limit what processes inside pods/containers can do (ex: process UID, linux capabilities, filesystem group,...)

ex of security context to prevent containers to ru their process as root :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - image: nginx
    name: nginx
```

set les credentials d'un user : `kubectl config set-credentials [-h]`

#### Network Policies

- an empty selector will match everything -> ex: `spec.podSelector: {}` will apply the policy to all pods in the current namespace
- selectors can only select pods in the same samespace as the NetworkPolicy
- all traffic to/from a pod is allowed until a policy is applied to it
- no deny rules in network policies : deny by default, accept explicitly
- if a network policy matches a pod but has a null rule, all traffic is blocked
- if there is *at least one* NetworkPolicy with a rule *allowing* the traffic, it means the traffic will be routed to the pod *regardless of the policies blocking* the traffic.
- the elements in the list of sources in the `from` rule of `ingress` are combined using a logical OR

ex of a "deny all" policy for all pods with label "app: web" :

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: web-deny-all
spec:
  podSelector:
    matchLabels:
      app: web
  ingress: []
```

ex of a policy that allows ingress from pods labeled "type: tool" only:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-tools
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - podSelector:
        matchLabels:
          type: tool
```

ex of a policy that allows all ingress traffic in the same namespace and from all pods labeled `app: db` in the namespaced labeled `team: one`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}
    - from:
        - namespaceSelector:
            matchLabels:
              team: one
          podSelector:
            matchLabels:
              app: db
```

#### Authentication & authorization (create a new user)

##### Authentication

- create and configure a user on the machine

  ```sh
  # create the user
  sudo useradd -s /bin/bash DevDan
  sudo passwd DevDan
  ```
- generate a private key and a certificate signing request for the user

  ```sh
  openssl genrsa -out DevDan.key 2048
  touch $HOME/.rnd
  openssl req -new -key DevDan.key -out DevDan.csr -subj "/CN=DevDan/O=development"
  ```
- generate a certificate for the user using their request and the K8s cluster's CA keys

  ```sh
  sudo openssl x509 -req -in DevDan.csr \
  	-CA /etc/kubernetes/pki/ca.crt \
  	-CAkey /etc/kubernetes/pki/ca.key \
  	-CAcreateserial -out DevDan.crt -days 45
  ```
- update the kube config file to add the new user and their key + certificate

  ```sh
  kubectl config set-credentials DevDan \ 
  	--client-certificate=/home/ubuntu/DevDan.crt \
  	--client-key=/home/ubuntu/DevDan.key
  ```
- create a context in the kube config file

  ```sh
  k config set-context DevDan-context \
  	--cluster=kubernetes \
  	--namespace=development \
  	--user=DevDan
  ```

\-> difference in kube config file :

```sh
ubuntu@k8s-control:~$ diff cluster-api-config .kube/config
9a10,14
>     namespace: development
>     user: DevDan
>   name: DevDan-context
> - context:
>     cluster: kubernetes
15a21,24
> - name: DevDan
>   user:
>     client-certificate: /home/ubuntu/DevDan.crt
>     client-key: /home/ubuntu/DevDan.key
```

\-> use the new context with kubectl :

```sh
$ k --context=DevDan-context get po
Error from server (Forbidden): pods is forbidden: User "DevDan" cannot list resource "pods" in API group "" in the namespace "development"
```

=> RBAC not configured yet

##### Authorization (RBAC)

- create a role with the user's permissions

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: development
    name: developer
  rules:
  - apiGroups: ["", "extensions", "apps"]
    resources: ["deployments", "replicasets", "pods"]
    verbs: ["list", "get", "watch", "create", "update", "patch", "delete"]
  ```

  :information_source: use `["*"]` for all verbs
- create a RoleBinding to associate the new role with the user

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: developer-role-binding
    namespace: development
  subjects:
  - kind: User
    name: DevDan
    apiGroup: ""
  roleRef:
    kind: Role
    name: developer
    apiGroup: ""
  ```

\-> test the context :

```sh
$ kubectl --context=DevDan-context run test --image nginx
pod/test created
$ kubectl auth can-i create deployments --namespace development --as DevDan
yes
```

or test RBAC using user impersonation :

```sh
kubectl auth can-i create deployments --namespace prod
kubectl auth can-i list secrets --namespace dev --as dave
```

### Ch16 : High availability

higher redundancy and fault tolerance with multiple control plane nodes with collocated databases
\-> 3 instances are required for etcd to be able to determine quorum if there is a problem with the data

\-> 2 possibilities for database placement :

- collocate the database with control planes -> join a kubeadm cluster as a control plane node : add `--control-plane` and a `certificate-key` to the join request
- use an external etcd database cluster -> takes more time & config ; set etcd external, endpoints and certificate location in `kubeadm-config.yaml`

depuis le control plane, créer un nouveau certificat pour join en tant que control plane au lieu de worker : `sudo kubeadm init phase upload-certs --upload-certs`

puis sur le nouveau node qui va join en control plane :

```sh
sudo kubeadm join k8scp:6443 \
--token ... --discovery-token-ca-cert-hash sha256:... \
--control-plane --certificate-key <clé récupérée avant>
```

#### Deploy highly-available cluster (proxy + multiple CP)

- deploy the proxy on one node (install for example HAProxy)
- install kube stuff on the new cp nodes (without kubeadm/join)
- configure `/etc/hosts` on every machine to match `k8s-control` (or chosen hostname) to the proxy's ip
- join control plane nodes
  *on the first control plane node*
  - create a token to be used for joining the cluster : `sudo kubeadm token create`
  - create a new ssl hash ([doc](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes)) :

    ```sh
    openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
       openssl dgst -sha256 -hex | sed 's/^.* //'
    ```
  - create a new control plane certificate for the join command :

    ```sh
    sudo kubeadm init phase upload-certs --upload-certs
    ```

  *on the new control plane nodes*
  - use kubeadm join :

    ```sh
    sudo kubeadm join k8s-control:6443 \
    --token REDACTED --discovery-token-ca-cert-hash sha256:redacted \
    --control-plane --certificate-key REDACTED \
    --cri-socket unix:///var/run/cri-dockerd.sock
    ```

    :warning: requires the cluster to have been initialized with `--control-plane-endpoint`

check etcd endpoint status (from an etcd pod, in a multi-cp cluster)
\-> endpoints = node ips + 2379 port

```sh
ETCDCTL_API=3 etcdctl -w table \
--endpoints 10.128.0.66:2379,10.128.0.24:2379,10.128.0.30:2379 \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key \
endpoint status
```

:information_source: setting up a load balancer is not required on the CKA exam

ex of setup commands :

```sh
sudo kubeadm init --control-plane-endpoint "k8scp:6443" --upload-certs --cri-socket unix:///var/run/cri-dockerd.sock --pod-network-cidr 192.168.0.0/16


cp nodes join :
sudo kubeadm join k8scp:6443 --token fhu1du.fpmxbcvmpaov09ku \
	--discovery-token-ca-cert-hash sha256:b88b17ff7ad450b0e804d1a5860afce29a734723e104c701b124c42a00f4ecab \
	--control-plane --certificate-key f3a0f59d806e265ab3f0e8d453fb8912037d2fde5b6a799502dea695216b1bc4 \
	--cri-socket unix:///var/run/cri-dockerd.sock

worker nodes join :
sudo kubeadm join k8scp:6443 --token fhu1du.fpmxbcvmpaov09ku \
        --discovery-token-ca-cert-hash sha256:b88b17ff7ad450b0e804d1a5860afce29a734723e104c701b124c42a00f4ecab \
		--cri-socket unix:///var/run/cri-dockerd.sock


# in an etcd pod
ETCDCTL_API=3 etcdctl -w table \
 --endpoints 10.0.0.6:2379,10.0.0.8:2379,10.0.0.5:2379 \
 --cacert /etc/kubernetes/pki/etcd/ca.crt \
 --cert /etc/kubernetes/pki/etcd/server.crt \
 --key /etc/kubernetes/pki/etcd/server.key \
 endpoint status
```

## Certified Kubernetes Administrator (cloud guru)

### Important notes

K8s names cannot contain uppercase letters :

> a lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters, '-' or '.'

exécuter une commande en boucle dans un container :

```yaml
command: ['sh', '-c', "while true; do echo Bonjour Hi >> /output/out.txt ; sleep 10; done"]

# doc version
command: ["/bin/sh"]
args: ["-c", "while true; do echo hello; sleep 10;done"]
```

- obtenir la liste des arguments (?) possibles d'un manifeste d'une ressource (ex clusterrole) : `kubectl explain clusterrole --recursive`
- obtenir la version de l'API à utiliser pour une certaine ressource : ^ ou `kubectl api-resources |grep clusterrole`

### Ch2: Getting started

#### Architecture overview

control plane : plusieurs composants, peut être réparti sur plusieurs serveurs (souvent machines dédiées)
\-> composants principaux :

- **kube-api-server** : expose l'API k8s, interface principale pour interagir avec le cluster
- **etcd** : "backend" datastore pour l'API, par rapport à l'état du cluster
- **kube-scheduler** : assigne les containers aux worker nodes
- **kube-controller-manager** : manage automation-related utility processes
- **cloud-controller-manager** : interface entre k8s et les plateformes cloud

<img src="C:\\Users\\EmmaNeiss\\AppData\\Roaming\\Typora\\typora-user-images\\image-20230426164843577.png" alt="image-20230426164843577" style="zoom: 33%;" />

nodes : là où les containers managés par k8s vont être exécutés
\-> composants principaux :

- **kubelet** : agent k8s qui communique avec le control plane, report le statut des containers au control plane, s'assure que les containers run comme demandé par le control plane
- container runtime (séparé de k8s) : le software qui run les containers, souvent docker/containerd
- kube-proxy : proxy réseau pour le networking entre les containers et les services du cluster

<img src="C:\\Users\\EmmaNeiss\\AppData\\Roaming\\Typora\\typora-user-images\\image-20230426165508252.png" alt="image-20230426165508252" style="zoom:33%;" />

<img src="C:\\Users\\EmmaNeiss\\AppData\\Roaming\\Typora\\typora-user-images\\image-20230426165533764.png" alt="image-20230426165533764" style="zoom:33%;" />

#### Building a Kubernetes cluster

Kubeadm : simplifie le setup du cluster k8s

étapes setup cluster :

- (config réseau/hostnames pour que les différents serveurs puissent communiquer entre eux)
- installer le container runtime sur tous les serveurs (même control plane)

##### Config steps

**for all nodes:**

- enable kernel modules for docker engine / containerd

```sh
# enable au lancement du serveur
cat << EOF | sudo tee /etc/modules-load.d/containerd.conf
> overlay
> br_netfilter
> EOF
# enable les modules tout de suite
sudo modprobe overlay
sudo modprobe br_netfilter
```

- config sysctl params

```sh
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
# apply parameters immediately
sudo sysctl --system
```

- install docker

```sh
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg lsb-release apt-transport-https

sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
sudo apt-get update

# delete version string
VERSION_STRING=5:23.0.1-1~ubuntu.20.04~focal
sudo apt-get install -y docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin
```

- setup containerd config

```sh
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
```

- disable swap (**required for k8s**)

```sh
sudo swapoff -a
```

- install k8s stuff (cf [doc](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/))

```sh
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# install the same version for all tools
sudo apt-get update && sudo apt-get install -y kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00
# disable automatic update
sudo apt-mark hold kubelet kubeadm kubectl
```

**control plane only:** setup the cluster

- init the cluster

```sh
sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.24.0

# setup kube config with previous command output
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- configure networking plugin (pour passer de STATUS `NotReady` à `Ready` pour le node `control-plane`)

```sh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

join les nodes :

```sh
kubeadm token create --print-join-command
# puis paste ça dans chaque worker node
```

=> somehow broken

**version pas broken**

```sh
    3  sudo apt-get update
    4  sudo install -m 0755 -d /etc/apt/keyrings
    5  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    6  echo   "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    7    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    8  sudo apt-get update
    9  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   10  sudo modprobe overlay
   11  sudo modprobe br_netfilter
   12  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   13  net.bridge.bridge-nf-call-iptables  = 1
   14  net.bridge.bridge-nf-call-ip6tables = 1
   15  net.ipv4.ip_forward                 = 1
   16  EOF
   17  sudo sysctl --system
   18  sudo usermod -aG docker $USER
   19  logout
   20  sudo systemctl status docker
   21  sudo sed -i 's/disabled_plugins/#disabled_plugins/' /etc/containerd/config.toml
   22  docker ps -a
   23  sudo systemctl restart containerd
   24  sudo swapoff -a
   25  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
   26  cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOFS
   27  sudo apt-get update && sudo apt-get install -y kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00
   28  sudo apt-mark hold kubelet kubeadm kubectl
   29  clear
   30  sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.24.0
   34  mkdir -p $HOME/.kube
   35  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   36  sudo chown $(id -u):$(id -g) $HOME/.kube/config
   39  kubectl get no
   40  kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

#### Namespaces

namespace: "cluster" virtuel (séparation logique des objets du cluster) -> pratique pour séparer plusieurs apps qui tournent sur un cluster, ou pour permettre à plusieurs teams d'utiliser le même cluster physique

```sh
kubectl get namespaces
# shortcut
k get ns
kubectl create namespace <my-namespace>
```

### Ch3: K8s Management

#### High availability in K8s

**multiple control plane nodes**, multiple instances of each control plane component
\-> **load balancer** to balance traffic to the multiple control plane nodes/kube-api server instances
\-> à la fois les outils comme kubectl et les instances de kubelet des worker nodes vont passer par le load balancer

<img src="C:\\Users\\EmmaNeiss\\AppData\\Roaming\\Typora\\typora-user-images\\image-20230501104619224.png" alt="image-20230501104619224" style="zoom: 50%;" />

plusieurs design patterns pour etcd:

- *stacked etcd* : etcd tourne sur les mêmes servers/nodes que le reste des composants du control plane (pattern utilisé par **kubeadm**) -> **chaque control plane node** va avoir **sa propre instance de etcd**
- *external etcd*: etcd tourne sur des nodes séparés du reste du control plane -> découplage du nombre de nodes pour etcd et pour le reste des composants du control plane

#### K8s management tools

- kubectl : official CLI for k8s
- kubeadm : tool for quickly & easily create k8s clusters
- minikube : ~kubeadm mais pour setup un serveur sur une seule machine
- helm : templating/package management for k8s objects
- kompose : transitioning from docker to k8s (convert docker-compose files to k8s objects)
- kustomize : config management tool (~helm dans le sens templating)

#### Safely draining a k8s node

draining a node : gracefully terminate containers running on the node (+ potentially move containers to another node)

```sh
kubectl drain <node-name> [--ignore-daemonsets]
```

uncordoning a node : signal to k8s that the node can be used to run pods

```sh
kubectl uncordon <node-name>
```

\-> pas possible de drain un node qui a des pods non managés par des replicaset/stuff similaire (genre pods déployés en tant que pods directement) (utiliser `--force` pour delete le pauvre petit pod, sera pas rescheduled)

**uncordoning** does **not** automatically **rebalance** deployment pods

#### Upgrading k8s with kubeadm

control plane node upgrade steps :

- drain control plane node
- upgrade kubeadm package on the control plane node

  ```sh
  sudo apt-get update
  # optionnel: en cas de problème de clé pur le repo k8s
  curl -L https://dl.k8s.io/apt/doc/apt-key.gpg | sudo apt-key add -
  
  sudo apt-get install --allow-change-held-packages kubeadm=<version>
  # note: version sous forme x.xx.x-xx (ex: 1.22.2-00)
  ```
- plan + apply upgrade (`kubeadm upgrade plan [1.22.2]` et `kubeadm upgrade apply [1.22.2]`)
- upgrade kubelet & kubectl packages (package manager)
- si jamais changement aux service files avec l'update : `sudo systemctl daemon-reload` et `sudo systemctl restart kubelet`
- uncordon node

worker node upgrade steps :

- drain node *(from the control plane node)*
- upgrade kubeadm package (via system package manager)
- upgrade kubelet config (`kubeadm upgrade node`)
- upgrade kubelet & kubectl packages (system package manager)
- daemon reload & restart kubelet
- uncordon node *(from the control plane node)*

#### Backing up and restoring etcd data

etcd is k8s's backend that stores data for k8s objects, apps, configs,...
\-> can be backed up using `etcdctl` with `etcdctl snapshot save`
\-> restore with `etcdctl snapshot restore <backup_file>`

- get le nom du cluster etcd (pour voir si les données sont bien restorées ensuite en comparant avant/après) :

```sh
# ETCDCTL_API=3 nécessaire pour spécifier la bonne version de etcdctl (possible d'export pour le reste des commandes aussi -> a l'air borked)
ETCDCTL_API=3 etcdctl get cluster.name \
  --endpoints=https://10.0.1.101:2379 \
  --cacert=/home/cloud_user/etcd-certs/etcd-ca.pem \
  --cert=/home/cloud_user/etcd-certs/etcd-server.crt \
  --key=/home/cloud_user/etcd-certs/etcd-server.key
```

- backup les données du cluster etcd :

```sh
ETCDCTL_API=3 etcdctl snapshot save /home/cloud_user/etcd_backup.db \
  --endpoints=https://10.0.1.101:2379 \
  --cacert=/home/cloud_user/etcd-certs/etcd-ca.pem \
  --cert=/home/cloud_user/etcd-certs/etcd-server.crt \
  --key=/home/cloud_user/etcd-certs/etcd-server.key
```

- vérifier quelques infos sur le backup pour s'assurer qu'il soit ok :

```sh
ETCDCTL_API=3 etcdctl --write-out=table snapshot status etcd_backup.db
```

- brutaliser etcd :skull_and_crossbones: :

```sh
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd
```

- restorer backup avec données au bon endroit

```sh
sudo ETCDCTL_API=3 etcdctl snapshot restore /home/cloud_user/etcd_backup.db \
  --initial-cluster etcd-restore=https://10.0.1.101:2380 \
  --initial-advertise-peer-urls https://10.0.1.101:2380 \
  --name etcd-restore \
  --data-dir /var/lib/etcd
```

- relancer etcd

```sh
sudo systemctl start etcd
```

### Ch4: K8s Object Management

#### Working with kubectl

kubectl commands :

- `get <object_type> [object_name]` : list objects in the cluster
- `describe <object type> <object_name>` : detailed info about objects
- `create [-f file_name]` : create an object (with yaml or imperative) ->:information_source: will fail if object already exists (type/name/namespace identical)
- `apply -f file_name` : like create, but will update existing objects
- `delete` : delete existing object
- `exec <pod_name> [-c container_name] -- command` : execute command inside a container in a pod

imperative commands : using command line flags instead of yaml files
\-> ex: simple nginx deployment : `kubectl create deployment my-nginx-depl --image=nginx`

quick sample yaml : use `--dry-run -o yaml`, ex: `kubectl create deployment sample-deploy --image=nginx --dry-run -o yaml`
or find some examples in the k8s doc

record a command used to modify a resource (visible in the resource's annotations) : `--record` after a command (e.g. `kubectl scale deployment my-depl replicas=5 --record`)

#### Managing K8s RBAC

RBAC = role-based access control

roles & clusterroles define a set of permissions
(cluster)rolebindings connect (cluster)roles to users (or groups, or service accounts)
\-> :information_source: a RoleBinding can reference a ClusterRole to grant permissions defined in that ClusterRole to resources inside the RoleBinding's namespace (useful for set of common rules across apps,...)

tester les permissions d'un user (à partir de sa kubeconfig) : `kubectl get pods -n my-namespace --kubeconfig the-user-config`

#### Creating service accounts

service account = used by container processes (within pods) to authenticate with the k8s api -> if pods need to communicate with the k8s api

#### Inspecting pod resource usage

k8s metrics server : addon that collects data about resource usage
\-> data can be viewed for each node/pod with `kubectl top <pod|node> [--sort-by <json_path>] [--selector <selector>]`

addon can be installed by applying its k8s manifest : `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`

check if the metrics server is responsive: `kubectl get --raw /apis/metrics.k8s.io/`

ex: obtenir le nom du pod qui consomme le + de CPU dans le namespace `beebox-mobile` avec le label `app=auth` :

```sh
kubectl top po -n beebox-mobile --selector app=auth --sort-by cpu --no-headers |head -n 1 |awk '{print $1}' > cpu-pod-name.txt
```

### Ch5: Pods & containers

#### Application configuration

2 ways to store config data in k8s :

- **ConfigMap** : key-value map (can be nested,... -> yaml format)
- **Secrets** : similar to ConfigMap but can store sensitive data

\-> these values can be passed to containers as

- **environment variables** :

```yaml
spec:
  containers:
  - ...
    env:
    - name: MYENVVAR
      valueFrom:
        configMapKeyRef:
          name: my-configmap
          key: mykey1
    - name: MYSECRETENVVAR
      valueFrom:
        secretKeyRef:
          name: my-secret
          key: secretkey1
```

- **configuration volumes** (mounted volumes in the container file system) -> each top-level key will be a file containing all keys below that level, ex :

```yaml
spec:
  containers:
  - ...
    volumeMounts:
    - name: configmap-vol
      mountPath: /etc/config/configmap
    - name: secret-vol
      mountPath: /etc/config/secret
  volumes:
  - name: configmap-vol
    configMap:
      name: my-configmap
  - name: secret-vol
    secret:
      secretName: my-secret
```

Note: a Secret must be base64-encoded
\-> use `base64` to encode in bash (ex `echo -n 'verysecret' | base64`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  secretKey1: <base64_encoded_value>
```

créer un secret avec le contenu d'un fichier (ex htpasswd) :

```sh
kubectl create secret generic nginx-htpasswd --from-file .htpasswd
```

#### Managing container resources

**resource requests** : expecting resource (CPU, memory) consumption for a container -> helps for **pod scheduling** (only)
\-> CPU unit : "m" = 1/1000 of 1 CPU
\-> memory unity : bytes (ex: Mi)
:information_source: *Note:* The scheduled pod with a resource request will **remain pending** until the scheduler finds a node capable of providing the pod with the requested resources *\-> might never happen!*

**resource limits** : prevent containers from using more than the limit
\-> container runtime enforces the limit, behavior may vary (terminate container, throttle process)

resource requests & limits ex:

```yaml
...
containers:
- name: busybox
  image: buybox
  resources:
    requests:
      cpu: "250m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "256Mi"
```

#### Monitoring container health with probes

- **liveness probe** : determine whether a container app is in a healthy state ; run constantly on a schedule -> detection mechanism can be customized

```yaml
...
containers:
- name: busybox
  image: busybox
  command: ['sh', '-c', 'while true; do sleep 3600; done']
  livenessProbe:
    exec:		# one type of liveness probe
      command: ["echo", "Hello, World"]
    initialDelaySeconds: 5
    periodSeconds: 5
- name: nginx
  image: nginx
  livenessProbe:
    httpGet:	# probe for http services
      path: /
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 5
```

- **startup probes** : like liveness probes but run at container startup (stop running once they succeed) -> detect if an app started successfully

```yaml
...
containers:
- name: nginx
  image: nginx
  startupProbe:
    httpGet:
      path: /
      port: 80
    failureThreshold: 30
    periodSeconds: 10
```

- **readiness probe** : determine when a container is ready to accept requests -> prevent user traffic from being sent to pods whose containers are not ready yet

```yaml
...
containers:
- name: nginx
  image: nginx
  readinessProbe:
    httpGet:
      path: /
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 5
```

:information_source: *Note:* when using a HTTP request probe, pods will be considered healthy if the code returned is between 200 and 400

#### Building self-healing pods with restart policies

restart policies control what happens when probes detect a container is unhealthy
\-> 3 policies:

- Always : container will be restarted when it is unhealthy, fails, or stops (even if completed successfully) *(default)* -> for apps that should always be running
- OnFailure : restart container only if becomes unhealthy or exits with an error code -> for apps that need to run once successfully then stop
- Never -> for apps that should be run once but never automatically restarted

#### Creating multi-container pods

containers share resources like network & storage
\-> best practice : keep containers in separate pods unless they need to share resources

containers in the same pod can communicate with one another on any port, even if this port is not exposed to the cluster
they can also share data in the pod's storage with volume mounts

secondary container for a first one (e.g. to add functionalities) : *sidecar*

:information_source: when `kubectl get pods`, the `READY` numbers are the number of ready *containers* of the pod

#### Init containers

init containers run once (in order) to completion during the startup of a pod before app containers run

```yaml
# Pod spec
spec:
  containers:
  - ...
  initContainers:
  # will sleep for 30 seconds before the main container runs
  - name: delay
    image: busybox
    command: ['sleep', '30']
  
  # will wait for another k8s service to exist before completing
  - name: delay-startup
    image: busybox:1.27
    command: ['sh', '-c', 'until wget shipping-svc; do echo waiting for shipping-svc; sleep 2; done']
```

### Ch6: Advanced Pod Allocation

#### K8s Scheduling

what the k8s scheduler takes into account when selecting a node for a pod :

- resource request vs available node resources
- various config to influence scheduling
  \-> ex: `nodeSelector` limits which nodes the pod can be scheduled on (using labels)

  ```yaml
  ...
  containers:
  - image: nginx
    name: nginx
  nodeSelector:
    myLabel: myValue		# pod scheduled on nodes labeled this way only
    # ex:
    disktype: ssd
  ```

  \-> or `nodeName` : bypass the scheduling and assign a pod to a specific node instead

  ```yaml
  containers:
  ...
  nodeName: k8s-worker1
  ```

add a label to a node : `kubectl label nodes <node_name> myLabel=myValue`

#### Using DaemonSets

a daemonset automatically runs a copy of a pod on each node, including the new nodes added to the cluster

daemonsets respect normal scheduling rules (labels, taints, tolerations) -> will not be created if scheduling conditions not met

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-ds
spec:
  # allows the daemonset to specify which pods are managed by the daemonset
  selector:		
    matchLabels:
      app: my-daemonset
  template:	# pod template
    metadata:
      labels:
        app: my-daemonset	# must match matchLabels ^
    spec:
      containers:
      - ...
```

\-> ex d'un daemonset qui va delete régulièrement le contenu de `/etc/beebox/tmp` sur tous les nodes hôtes :

```yaml
apiVersion: apps/v1		# IMPORTANT: v1 tout seul marche pas
kind: DaemonSet
metadata:
  name: cleanup-ds
spec:
  selector:
    matchLabels:
      app: cleanup
  template:
    metadata:
      labels:
        app: cleanup
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'while true; do rm -rf /beebox-tmp/; sleep 60; done']
        volumeMounts:
        - name: beebox-tmp
          mountPath: /beebox-tmp
      volumes:
      - name: beebox-tmp
        hostPath:
          path: /etc/beebox/tmp
```

#### Using Static Pods

**static pod** = pod **managed directly by kubelet** on a node, not by the k8s API server -> can run even if no k8s api server is present

kubelet automatically creates static pods from manifests located in the manifest path on its node (by default, `/etc/kubernetes/manifests/`)

kubelet also create a **mirror pod** for each static pod, allowing to see the static pod status via the k8s api, without being able to change it via the api (has to be managed by kubelet directly) (if deleted, will be recreated instantly by the node's kubelet)

### Ch7: Deployments

#### K8s deployments overview

deployment : defines a desired state for a replicaset (multiple copies of the same pod)
\-> includes : `replicas` (nb of pods), `selector` (identifies pods managed by the deployment), `template` (defines pods created by the replicaset)

deployments to :

- easily scale up/down an app by changing the nb of replicas
- perform rolling updates to deploy a new software version (without downtime)
- roll back to a previous software version

:warning: deployments are part of the `apps/v1` API (or `extensions/v1beta1` ?)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-deployment
  template:		# pod template
    metadata:	# deployment will manage the name for its pods
      labels:
        app: my-deployment		# must match the selector
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.1
        ports:
        - containerPort: 80
```

:information_source: all pods that match the selector of the deployment will be managed by the deployment, even if they were created differently

#### Scaling Applications with Deployments

deployment replicas for horizontal scaling
\-> update that number :

- in the YAML descriptor (then `kubectl apply`)
- using `kubectl edit deployment my-deployment` directly (*note:* use ` --save-config` to add an annotation to save the change)
- `kubectl scale deployment.v1.apps my-deployment --replicas=10`

#### Managing Rolling Updates with Deployments

rolling update: make changes to a deployment's pods at a controlled rate, gradually replacing old pods with new pods -> no downtime
\-> rolling updates can be done with any changes to the template of the pods in the deployment
\-> rolling updates are automatically triggered by changes to the pod template (`kubectl apply`, `kubectl edit`, `kubectl set image deployment/my-deploy nginx-container=nginx:1.20.0 [--record]`)

rollback : similar to rolling update but with the old version
\-> `kubectl rollout undo deployment/my-depl` to rollback the last revision (or `--to-revision=1` to rollback to a specific revision)

check the last rolling update's status: `kubectl rollout status deployment/my-depl`

check rollout history: `kubectl rollout history deployment/my-depl`

:information_source: `--record` is deprecated, to keep a clean rollout history use `kubectl annotate <resource>/<name> kubernetes.io/change-cause="image update to 1.2.1" [--overwrite=true]`

### Ch8: Networking

#### K8s Networking Architectural Overview

k8s network model : standards for networking between pods
\-> different implementations, ex Calico network plugin

networking standards:

- each pod has its own unique IP address within the cluster
- any pod can reach any other pod using its IP address (virtual network) -> no need to be aware of nodes

#### CNI plugins overview

CNI plugins (Container Network Interface): k8s network plugins for network connectivity between pods according to k8s's standard

choose a CNI plugin: check the k8s doc, depending on the use case
\-> installing network plugins: process depends on the plugin

a CNI plugin must be installed to be able to deploy pods on the k8s cluster (nodes will remain NotReady until a plugin is installed)

#### K8s DNS

k8s virtual network includes a DNS, runs as a pod within the cluster (`kube-system` namespace)
\-> kubeadm uses CoreDNS as the DS solution

all pods are automatically given domain names like: `<pod-ip>.<namespace>.pod.cluster.local`

troubleshooting des problèmes de nodes NotReady liés au network plugin : `kubectl describe no <node> |grep -i cni`
\-> aussi regarder `Non-terminated pods` pour trouver des pods qui ont pas pu start (ex kube-proxy ici)

```yaml
$ kubectl describe no <node>
...
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  Ready            False   Thu, 04 May 2023 15:08:01 +0000   Thu, 04 May 2023 14:42:17 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
 
```

#### Using Network Policies

network policy: allows to control the flow of network communications to/from pods
\-> more secure cluster network, keep pods isolated from traffic they do not need

:warning: by **default**, pods are **non-isolated** and completely open to all communication as long as no network policy selects them
\-> if **any network policy selects a pod**, it is then considered **isolated** and will **only** be open to traffic **explicitly allowed** by NetworkPolicies

network policies can apply to ingress and/or egress traffic, and can specifies entities that are either **pods**, **namespaces** or **IP blocks**
\-> use selectors with pods/namespaces and cidr for ip blocks

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-nw-policy
spec:
  podSelector:		# matches pods in the nw policy's namespace
    matchLabels:
      role: db
  
  # define isolation type to be applied to selected pods
  # if no rules are specified for a given type, all traffic will be blocked
  policyTypes:		
    - Ingress
    - Egress
  
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: db
    ports:
      - protocol: TCP
        port: 80

  egress:
    - to:
        - ipBlock:
            cidr: 10.0.2.0/24
    - namespace:
        matchLabels:
          app: db
```

selectors to use with `from` and `to` :

- `podSelector` (e.g. with `matchLabels`)
- `namespaceSelector`
- `ipBlock` (cidr blocks)

specify ports to allow traffic for these ports only

:information_source: label a namespace (or any object ?) : `kubectl label namespace <ns-name> foo=bar`

network policies do not conflict, they are **additive** :

> For a connection from a source pod to a destination pod to be allowed, both the egress policy on the source pod and the ingress policy on the destination pod need to allow the connection. If either side does not allow the connection, it will not happen.

### Ch9: Services

#### K8s Services Overview

**service**: expose an app running as a set of pods -> abstraction over pods ; service routes traffic to its pods in a load-balanced fashion

**endpoints**: backend entities to which services route traffic -> each pod has an endpoint associated with its service

:information_source: determine which pods a service routes traffic to: look at the service's endpoints
\-> `kubectl get endpoints <service-name>`

#### Using K8s services

service types :

- `ClusterIP` (default) : expose apps **inside the cluster** -> use when clients are other pods within the cluster
- `NodePort` : expose apps **outside** the cluster network (listens on a specific port of all cluster nodes) -> use when clients will be accessing the service from outside the cluster
- `LoadBalancer` : expose apps **outside** the cluster, but use and **external cloud load balancer** -> only works with **cloud platforms** that include load balancing
- `ExternalName` (outside CKA scope)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  type: ClusterIP
  selector:
    app: svc-example
  ports:
    - protocol: TCP
      port: 80			# port the service is listening on
      targetPort: 8080	 # port the service is routing to on the target port
      
---
apiVersion: v1
kind: Service
metadata:
  name: nodeport-svc
spec:
  type: NodePort
  selector:
    app: svc-example
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080	# the port all nodes of the cluster listen on for this svc
```

:information_source: you can have any number of service routing to the same pods

:information_source: busybox with curl: `image: radial/busyboxplus:curl`

#### Discovering K8s services with DNS

the k8s DNS assigns DNS FQDN to services of the following format: `<svc-name>.<namespace>.svc.<cluster-domain>` with the default cluster domain being `cluster.local`

\-> a service's FQDN can be used to reach the service from within **any namespace** in the cluster (like the IP address)
\-> pods within the **same namespace** can also use the **shortened** version : `<service-name>`

#### Managing Access from Outside with K8s Ingress

Ingress: manages external access to **services** in the cluster, with more features than a simple NodePort service: SSL termination, advanced load balancing, name-based virtual hosting,...

ingress objects need an Ingress controller to do anything (different available)

ingresses define a set of **routing rules**, and each route has a set of **paths**, each with a **backend**
\-> requests matching a path will be routed to this path's associated backend

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - http:
      paths:
      - path: /something
        pathType: Prefix
        backend:
          service:
            name: my-svc
            port:
              number: 80
              # or name: web if named port
```

:information_source: if a service uses a named port, an Ingress can also use the port's name instead of the port number for routing

ex of named port:

```yaml
ports:
- name: web
  protocol: TCP
  port: 80
```

:information_source: make sure an ingress found the service it must be routing to: `kubectl describe ingress my-ingress` and in the `Rules` section, the backend should show its service without errors

ex d'ingress controller : https://kubernetes.github.io/ingress-nginx/deploy/#quick-start
\-> install :

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml
```

puis une fois l'ingress créé, créer un port-forward :

```sh
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8080:80
```

> Endpoints are the underlying entities (such as Pods) that a Service routes traffic to.

### Ch10: Storage

#### K8s Storage Overview

**container file systems** are **ephemeral** -> if the container is deleted/recreated, data stored in container file system is lost

**volumes** store data outside the container file system, while allowing the container to access this data at runtime
\-> data **persists** beyond the lifetime of the container

**persistent volumes** : more advanced form of volume, treat storage as an **abstract** resource that can be consumed by pods
\-> uses **persistent volume claims**

volume types (for volumes & persistentVolumes) : NFS, cloud storage, configmap/secrets, simple directory on the k8s node,...

\-> **volumes** are **coupled to a pod** via their lifecycle ; enables safe container restarts + sharing data between containers of the same pod

\-> **persistent volumes** decouples the storage from the pod, its lifecycle is independent ; enables safe pod restarts & sharing data between pods

#### Using K8s Volumes

regular volume setup :

```yaml
spec:
	containers:
	- name: busybox
      image: busybox
      volumeMounts: 	# mount the volume on a specific container
      - name: my-volume
        mountPath: /output
	# as part of the pod (not container!) spec:
    volumes:
    - name: my-volume
      hostPath:
        path: /data
```

\-> the **volume** is available for the whole **pod**, and the **volumeMount** is done for each **container** that uses the volume
\-> the same volume can be **mounted to multiple containers** in the same pod (mount path can be different)

common volume types :

- `hostPath`: stores data in a specified directory **on the host node**
- `emptyDir` : **temporary** volume that stores data in a dynamically created location on the node that **exists only as long as the pod exists on the node**
  \-> useful for **sharing data between containers** in the same pod
  \-> ex:

  ```yaml
  volumes:
  - name: my-emptyDir-volume
    emptyDir: {}		# no need to specify any data, just the key is needed
  ```

#### Project ConfigMap (or secret) onto files in the container

```yaml
    volumeMounts:
    - name: cfg
      mountPath: /var/dev-sdk
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: cfg
    configMap:
      name: devconfig
      items:
      - key: sdk.cfg
        path: config/sdk.cfg
      - key: language
        path: lang/golang/version/value.txt
```

#### K8s Persistent Volumes

persistent volumes : abstraction over storage (like memory/cpu)
\-> includes a set of attributes to describe the underlying storage resource (disk/cloud storage,...)

```yaml
apiVersion: v1		# not storage.k8s.io/v1 !
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  storageClassName: localdisk
  persistentVolumeReclaimPolicy: Recycle
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
  	path: /var/output
```

- `storageClass `: allows k8s admins to specify the types of storage services provided on their platform -> ex: admin could create storage classes called `slow` and `fast\`\`
- \`\`allowVolumeExpansion` (*storage class attribute*): determines whether or not volumes can be resized after creation
- `persistentVolumeReclaimPolicy` : determines how storage resources can be reused when the PV's claims are deleted 
  - `Retain` (data will remain in the storage, **manual cleanup required**),
  - `Delete` (automatically deletes the underlying storage resource, **cloud storage only**),
  - `Recycle` (automatically deletes all data in the underlying storage resource, allowing the PV to be reused)

ex of storageClass (automatically created if not already existing when creating a PV) :

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: localdisk
provisioner: kubernetes.io/no-provisioner
allowVolumeExpansion: true		# default: false
```

**PersistentVolumeClaim** : represents a user's request to use storage resources, defines a set of attributes (similar to PV attributes)
\-> when a PVC is created, it will look for a PV that meets the requested criteria ; if found, will automatically be bound to this PV

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: localdisk
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

\-> using a PV in a pod requires an existing PVC to use:

```yaml
containers:
- name: ...
  image: ...
  volumeMounts:
  - name: pv-storage
    mountPath: /output
  volumes:
  - name: pv-storage
    persistentVolumeClaim:
      claimName: my-pvc
```

**expanding** a PVC without interruption: edit `spec.resources.requests.storage` of the PVC, if the storage class supports resizing volumes and has `allowVolumeExpansion: true`

<img src="C:\\Users\\EmmaNeiss\\AppData\\Roaming\\Typora\\typora-user-images\\image-20230508184548577.png" alt="image-20230508184548577" style="zoom: 33%;" />

### Ch11: Troubleshooting

#### K8s cluster troubleshooting

- kube api server : if down, impossible to use kubectl to interact with the cluster -> similar errors can mean kubectl is not configured properly -> fixes : make sure `docker` / `containerd` and `kubelet` services are running on the control plane nodes
- node status: get/describe nodes ("Conditions" section of the describe)
- service status on a node (if a node has problems): systemctl status/enable for container runtime/kubelet
- checking system pods (*kubeadm clusters*) : `kubectl get pods -n kube-sytems`

#### Checking cluster & node logs

service logs: `journalctl -u kubelet/docker` (or `/var/log/kube-*` on the host, not for kubeadm clusters -> use `kubectl logs <pod>` for kubeadm clusters)

#### Troubleshooting applications

checking a pod's status: `kubectl get/describe po`

run commands inside a container : `kubectl exec <pod> [-c <container>] [-it] -- <command>`

#### Checking container logs

container logs: everything that the container outputs on stdout/stderr

`kubectl logs <pod> [-c <container>]`

#### Troubleshooting networking

kube-proxy or dns if networking issues
\-> pods in the kube-system namespace for kubeadm clusters

troubleshooting tool: run a container with the `nicolaka/netshoot` image

## K8s troubleshooting course (ACG)

updating a deployment: by default, deployments will make use of surge capacity to deploy new pods before deleting the old ones -> **nodes need capacity for update surge**

**daemonsets** schedule a pod on each node -> useful for logging/management tools
\-> use flags & taints for advanced scheduling (ex: different types of nodes which need different daemonsets)
\-> by default, daemonsets not scheduled on control plane nodes

statefulsets : schedule a specific number of replicas (like deployments) but maintain sticky identities for each pod ; pods have identical specs but are not interchangeable (assigned unique IDs that persist through rescheduling: network id and storage id)
\-> deployments are typically used for stateless apps, but can be used for stateful apps with a persistent volume
\-> statefulsets can be used to manage storage
\-> statefulsets require a headless service resource (specific )

headless services ([doc](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)):

> Sometimes you don't need load-balancing and a single Service IP. In this case, you can create what are termed "headless" Services, by explicitly specifying `"None"` for the cluster IP (`.spec.clusterIP`).

jobs: unlike deployments, they are meant to run, complete and end
\-> good for handling batch-processing tasks
\-> can run in parallel (:warning: app should be designed accordingly)

get le CPU alloué sur tous les nodes du cluster :

```sh
kubectl describe nodes | grep 'Name:\|Allocated' -A 5 | grep 'Name\|cpu'
```

get container termination error details:

```sh
kubectl get pod <pod-name> -o yaml
```

\-> look under `.status.containerStatuses`

\-> extract that info: (otherwise do some grep like `kubectl get pod puzzle-plaza -o yaml |grep terminated -A 5 |grep Error |uniq`)

```sh
kubectl get pod <pod-name> -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"
```

\-> `termination-log` = a log that is maintained after a pod has been terminated (see [docs](https://kubernetes.io/docs/tasks/debug/debug-application/determine-reason-pod-failure/))

- logs d'un pod qui a été terminated (pas sûr que ça marche toujours) :

```sh
kubectl logs pod_name --previous # or -p
```

- créer un daemonset qui run sur le control plane aussi :

```yaml
spec:
  selector:
    matchLabels:
      name: my-ds
  template:
    metadata:
      labels:
        name: my-ds
    spec:
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
```

#### Troubleshooting a running pod

- check container logs
- run commands inside the container
- attach a debug container (ephemeral container)

ephemeral containers : container that attach to the process namespace of a target
\-> ex: create a container that has a shell/a specific debugging tool and attach it to the pod that needs to be troubleshooted
\-> :warning: not fully available as of k8s 1.22 (need to enable ephemeral containers feature gate, not supported by all container runtimes)
\-> :warning: also not supported by static pods
\-> must be deleted manually

#### crictl

CRI : container runtime interface
\-> k8s's solution for openness and interoperability ; open standard

identify the runtime being used by a k8s cluster : `kubectl get nodes -o wide` returns container runtime

`crictl` : debugging tool for runtime issues with containers
\-> supports inspecting and creating pods in k8s clusters (:warning: changes will be overwritten by kubelet)
\-> commands map closely to docker cli

## Généralités

nodes = machines sur lesquelles les containers run
\-> 1 kublet par node, communique avec le control plane

container runtime : fait pas partie de k8s (ex: docker)

kube proxy : network proxy, pour le networking entre les containers et les services dans le cluster

namespaces : virtual clusters backed by the same physical cluster
\-> separate & organize resources
\-> `kube-system` : namespace utilisé pour les composants système (par ex control plane)

possible de lister les pods de tous les namespaces avec l'option `--all-namespaces`, ou juste un namespace particulier : `--namespace`ou `-n`

kubeadm : cluster setup tool

nodes communiquent avec le control plane via la k8s API

rendre un container accessible en dehors du réseau virtuel k8s : exposer le pod en tant que k8s service
\-> ex : `kubectl expose deployment hello-node --type=LoadBalancer --port=8080` (option `--type=LoadBalancer` pour exposer en dehors du réseau virtuel)

par défaut, les pods sont accessibles par d'autres pods/services du même cluster uniquement

> The containers in a Pod share an IP Address and port space, are always  co-located and co-scheduled, and run in a shared context on the same  Node.

un pod peut contenir plusieurs containers assez tightly-coupled ; toujours attaché à son node

\-> node = VM(/PM) managé par le control plane
\-> un node run au moins kubelet (communication node <-> control plane) + 1 container runtime (ex: docker)

each pod in k8s has its unique IP (even pods on the same node)

vérifier qu'un service est bien delete mais pas son/ses pods (minikube) :

`$ curl $(minikube ip):$NODE_PORT` devrait fail to connect (tentative d'accès depuis l'extérieur du cluster)
`$ kubectl exec -ti $POD_NAME -- curl localhost:8080` devrait fonctionner (exécution depuis un node du cluster)

pour le scaling d'un déploiement, changer le nombre de replicas du déploiement

`$ kubectl get pods -o wide` pour voir sur quel nodes les pods run

obtenir la commande pour join un cluster en tant que worker node : `kubeadm token create --print-joinn-command` (à run sur le serveur du control plane, le résultat sur les worker nodes)

obtenir rapidement un squelette de manifeste (par ex pour un déploiement) :

```bash
$ kubectl create deployment test-deployment --image=nginx --dry-run -o yaml > deployment.yaml #flag image nécessaire
```

\-> si jamais yaml récupéré depuis une ressource existante (`k get res foo -o yaml`) : pour utiliser ensuite, enlever :

- `creationTimestamp`
- `resourceVersion`
- `uid`
- `status` et tout ce qui vient après

flag `--record` pour garder une trace des changements sur les ressources k8s
\-> commande utilisée avec `--record` pour faire des modifications est enregistrée dans la partie annotations de `kubectl describe`

### High availability

utiliser plusieurs control planes pour une high availability du cluster -> load balancer pour envoyer le trafic vers le control plane

pattern "stacked etcd" : etcd run sur les mêmes nodes que le reste du control plane
// external etcd : nodes séparés pour etcd par rapport aux nodes du control plane
\-> kubeadm utilise le pattern stacked etcd

### K8s management tools

- kubectl : official cli
- kubeadm : pour créer des clusters facilement
- minikube : un peu comme kubeadm mais pour du single server/machine -> nice pour dev/automatisation
- helm : templating & package management for k8s objects
- kompose : pour transition entre docker et k8s

### Safely draining a k8s node

`kubectl drain <node name>` : draining = enlever temporairement un noeud du service (par ex pour de la maintenance)
\-> peut être nécessaire d'ignorer les daemonsets avec `--ignore-daemonsets` (k8s refuse de drain un node s'il a des pods daemonsets attachés)

`kubectl uncordon <node name>` pour autoriser à nouveau les pods à run sur le node après la maintenance
\-> ne va pas automatiquement rebalance la charge existante dessus

### RBAC

role : définit des permissions liées à un namespace // clusterrole : cluster-wide permissions
\-> rolebindings & cluusterrolebindings: lient des roles à des users

## Troubleshooting (misc)

### Kubectl not working - connection refused

> ```
> kubectl get no
> The connection to the server 172.31.112.193:6443 was refused - did you specify the right host or port?  
> ```

- kubectl pas configuré pour poke le bon endpoint sur un node autre que control plane : copier le contenu de ` /etc/kubernetes/admin.conf` du control plane dans `$HOME/.kube$config` sur le client
- vérifier que kube-api tourne (sur le bon port) :  `sudo lsof -Pni |grep 6443` ou `grep kube-apis`
  \-> vérifier que le process existe : `ps -ef |grep 6443`
  \-> si process existe mais pas dans lsof : reboot la machine
  \-> logs avec journalctl
  \-> logs plus détaillés mais de plein de trucs / vérifier si c'est pas la faute du swap qui est de retour : `less /var/log/syslog`

journalctl montre :

```
failed to destroy network for sandbox [...] plugin type=\"calico\"
```

=> **partie du problème  : firewall enabled**
\-> ubuntu:

```sh
sudo ufw status verbose
sudo ufw disable
```

\-> piste du problème : erreur de calico : https://github.com/projectcalico/calico/issues/4084

\-> autre piste : pb cgroups / compatibilité kernel

\-> raison pourquoi le cluster est tout broken : `Pod sandbox changed, it will be killed and re-created.`

~~\-> possible explication (~~`k describe no control-plane`) : `invalid capacity 0 on image filesystem` askip c'est chill

\-> autre pb : (`k describe po coredns-6d4b75cb6d-2fk4d -n kube-system`)

```
  Warning  FailedCreatePodSandBox  80m                 kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "d073e22e531d3274514ee5818646a2a19647d6c647a4f9ffcb27febe403e907c": plugin type="calico" failed (add): error getting ClusterInformation: Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": dial tcp 10.96.0.1:443: i/o timeout
```

- **autre source possible** : kube-apiserver du cluster est broken (ex: pb de configuration comme par ex mauvais argument dans le yaml à `/etc/kubernetes/manifests/kube-apiserver.yaml`)

  \-> aller voir avec `crictl ps` si le container de kube-apiserver est toujours up
  - si container là, check ses logs avec `crictl logs <container-id>`
  - si container pas là, aller voir les logs de kube-apiserver dans `/var/log/pods` ou `/var/log/containers/`
  - si pas de logs containers, aller voir les logs kubelet (`journalctl -xeu kubelet` ou `tail -f /var/log/syslog`)
    \-> manière plus opti de grep les logs :

    ```sh
    # lister les pbs de apiserver au fur et à mesure qu'ils arrivent
    tail -f /var/log/syslog |grep apiserver
    # si on cherche un pb de yaml
    tail -f /var/log/syslog |grep manifest
    ```

### Apt package update error (k8s repo)

erreur d'update des packages k8s de apt :

```sh
$ sudo apt-get update  
...
Get:5 https://packages.cloud.google.com/apt kubernetes-xenial InRelease [8993 B]                                                                                     Err:5 https://packages.cloud.google.com/apt kubernetes-xenial InRelease                                                                                                The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B53DC80D13EDEF05                                                  Fetched 181 kB in 1s (283 kB/s)                                                                                                                                      Reading package lists... Done                                                                                                                                        W: An error occurred during the signature verification. The repository is not updated and the previous index files will be used. GPG error: https://packages.cloud.google.com/apt kubernetes-xenial InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B53DC80D13EDEF05          W: Failed to fetch http://apt.kubernetes.io/dists/kubernetes-xenial/InRelease  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY B53DC80D13EDEF05                                                      
```

utiliser :

```
curl -L https://dl.k8s.io/apt/doc/apt-key.gpg | sudo apt-key add -
```

(update la clé gpg, cf [GH](https://github.com/kubernetes/k8s.io/pull/4837))

### Kubeadm cluster init error

erreurs creation du cluster avec kubeadm init :

```sh
kubeadm init --pod-network-cidr=10.1.1.0/24 --apiserver-advertise-address 10.0.0.6
[init] Using Kubernetes version: v1.27.2
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables does not exist
        [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
```

suivre les [prérequis](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#install-and-configure-prerequisites) !

### Kubectl not working

### Metrics server not working

Metrics server marche pas après déploiement : possiblement un problème de TLS pas setup (le cas par défaut)
\-> edit le déploiement metrics-server dans le namespace kube-system pour ajouter l'argument `--kubelet-insecure-tls` dans la liste des arguments du container

erreur:

```
"Failed to scrape node" err="Get \"https://192.168.2.212:10250/metrics/resource\": x509: cannot validate certificate for 192.168.2.212 because it doesn't contain any IP SANs
```

### Insufficient CPU on node with plenty of CPU

*problem*: node is chill with top (both `kubectl top no` and `top`), but pods are pending and can't be scheduled because of `Insufficient CPU` on a worker node
\+`kubectl describe no myworker` shows a limit above what would be used by the scheduled pod (`Allocated resources` in the node description, requests resources in pods description)

\-> see `kubectl get no myworker -o yaml`, `.status.allocatable.cpu` for the ACTUAL max CPU usage value

```sh
k get no -o json k8s-worker | jq .status.allocatable
```

\-> OR see "Limits" column : for example if CPU limit is `3 (150%)` then the actual scheduling limit is **2** (value for 100%) (same goes for requested column, pay attention to the %)

\-> other possible error : `cpu: "200Mi"` will be a valid value but will be blocked by limits !!

### Persistent Volume not deleting correctly

```
ubuntu@k8s-control:~$ k get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM        STORAGECLASS   REASON   AGE
pv-1   1Gi        RWX            Delete           Failed   smol/pvc-1                           14m
```

\-> possiblement plugin pour delete manquant :

```
$ k get pv -o yaml
...
status:
    message: 'error getting deleter volume plugin for volume "pv-1": no deletable volume plugin matched'
```

### DNS resolution issues

pods cannot resolve external hostnames :

```sh
# ubuntu "apt-get update"
W: Failed to fetch http://security.ubuntu.com/ubuntu/dists/jammy-security/InRelease  Temporary failure resolving 'security.ubuntu.com'

# busybox nslookup
ubuntu@k8s-control:~$ k exec -it busybox -- sh
/ #
/ # nslookup google.com
Server:         10.96.0.10
Address:        10.96.0.10:53

;; connection timed out; no servers could be reached

# wget
/ # wget google.com
wget: bad address 'google.com'
```

- vérifier que les pods coreDNS sont bien running : `kubectl get pods -n kube-system -l k8s-app=kube-dns`
- vérifier la config coreDNS dans sa configmap : `kubectl get configmap -n kube-system coredns -o yaml`
  \-> il faut soit un "forward" plugin soit un custom dns
- vérifier les logs de coreDNS : `k logs -n kube-system coredns-<smth>`
  \-> si `i/o timeout`, alors coreDNS ne peut pas reach le DNS (ici 10.0.0.2:53)

  ```
  [ERROR] plugin/errors: 2 google.com.eu-west-1.compute.internal. AAAA: read udp 10.0.0.236:34020->10.0.0.2:53: i/o timeout
  ```

=> **fix** (hacky but ok) : edit coredns configmap (`k edit -n kube-system configmaps coredns`) and replace :

```yaml
# <<<<<<<< OLD
forward . /etc/resolv.conf {
# >>>>>>>> NEW
forward . 8.8.8.8 {
```

\-> will use google's dns instead of local one

### Pod does CrashLoopBackOff for no apparent reason

no logs, no errors anywhere

\-> possible issue : container inside runsu no command and terminates immediatly
ex:

```sh
# CrashLoopBackOff
k run ubuntu --image=ubuntu
# ex:
ubuntu@k8s-control:~$ k run ubuntu --image=ubuntu
pod/ubuntu created
ubuntu@k8s-control:~$ k get po
NAME      READY   STATUS    RESTARTS       AGE
busybox   1/1     Running   48 (37m ago)   2d
ubuntu    1/1     Running   0              2s
ubuntu@k8s-control:~$ k get po
NAME      READY   STATUS      RESTARTS       AGE
busybox   1/1     Running     48 (37m ago)   2d
ubuntu    0/1     Completed   1 (3s ago)     5s
ubuntu@k8s-control:~$ k get po
NAME      READY   STATUS             RESTARTS       AGE
busybox   1/1     Running            48 (38m ago)   2d
ubuntu    0/1     CrashLoopBackOff   2 (22s ago)    37s

# runs normally
k run ubuntu --image=ubuntu -- sh -c 'sleep infinity'
```

### Service systemd broken, "[...] Start request repeated too quickly."

\-> erreur signifie que le service a die avant de pouvoir output de la verbose
=> problème core pour le service : sans doute permissions manquantes, fichier inexistant sur une partie core du service ou qqch comme ça
\-> d'abord, `systemctl stop <service>` puis `systemctl start <service>`
\-> check installed files : existing ? permissions ? location ?

### K8s ClusterIP service unreachable, connection refused

`Failed to connect to 10.106.209.43 port 80 after 0 ms: Connection refused` from cluster node and from a pod inside the container

- check that the container port is opened
- try to wget the target pod directly using busybox (and the pod's ip with `k get po -o wide`) -> if still not working, the container targeted might be broken

\-> **root cause here** : *don't* use `-- sh -c 'sleep 3600'` for an nginx image !!
=> **nginx pod troubleshooting tips** : `kubectl logs <nginx-pod>` **should return logs** if the container is running an nginx server as planned !!! no logs = no server = connection refused !

### Metrics server not working 'error: Metrics API not available'

`kubectl top po` returns `error: Metrics API not available` after applying the metrics server manifest

\-> possible issue : tls certificate not trusted and insecure tls not configured :
see metrics-server pod logs : if errors like below, tls issues :

```
E0614 14:43:29.881941       1 scraper.go:140] "Failed to scrape node" err="Get \"https://10.0.0.9:10250/metrics/resource\": x509: cannot validate certificate for 10.0.0.9 because it doesn't contain any IP SANs" node="k8s-worker"
```

=> fix :

```sh
kubectl -n kube-system edit deployments.apps metrics-server
# then add the following in spec.containers[0].args:
- --kubelet-insecure-tls
```

### Curl the k8s api via the kube proxy not working

\-> do not use the 6443 port to curl the api server, instead use the proxy port (8001 default ?)
\-> likewise, do not curl the machine hostname / eth0 ip, instead curl localhost:

```sh
curl http://10.0.0.5:8001/apis/metrics.k8s.io/v1beta1/nodes
curl: (7) Failed to connect to 10.0.0.5 port 8001 after 0 ms: Connection refused

curl  http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/nodes
{
  "kind": "NodeMetricsList",
```

if getting an error like `curl: (35) error:0A00010B:SSL routines::wrong version number`,
make sure to curl the http proxy endpoint

```sh
curl https://localhost:8001/api
curl: (35) error:0A00010B:SSL routines::wrong version number

curl http://localhost:8001/api
{
  "kind": "APIVersions",
...
```

### Node unable to join as control plane node with kubeadm (no stable controlPlaneEndpoint address)

```
kubeadm join ... --control-plane
[...]
error execution phase preflight:
One or more conditions for hosting a new control plane instance is not satisfied.

unable to add a new control plane instance to a cluster that doesn't have a stable controlPlaneEndpoint address
```

see [LF forum](https://forum.linuxfoundation.org/discussion/856782/lab-16-2-unable-to-join-second-master-node) : naming conflict between k8s cp master and proxy hostname
\-> change hostname in `/etc/hosts`

### Kubeadm init "kubelet-check initial timeout of 40s has passed"

```
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
```

\-> troubleshooting : use `journalctl -xeu kubelet.service` on this node to see the errors

possible explanation if using `--control-plane-endpoint` : kubelet cannot connect to the provided control plane endpoint
\-> check connectivity to the provided endpoint (possibly `/etc/hosts` misconfigured)

### Nginx Ingress throwing 404 on registered paths/hosts

~~mode details about the error :~~  **actually that's just normal**

```sh
kubectl describe ingress

Name:             simple
...
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  Sync    6m7s  nginx-ingress-controller  Scheduled for sync
```

\-> problème possible : manque l'annotation nginx `nginx.ingress.kubernetes.io/rewrite-target: /` sur l'Ingress

\-> autre problème possible : curl sans `-H "Host: www.host.com"` si host précisé dans la règle
(ex de curl pour `host: www.foo.com` : `curl 192.168.194.75/bar -H "Host: www.foo.com"` avec IP de pod d'ingress controller)

\-> autre possibilité : mauvais `ingressClassName` (utiliser `k get ingressclass` pour avoir le nom de la classe à utiliser)

### Nginx Ingress endpoints returning 301 errors

```
controlplane $ curl world.universe.mine:30080/asia 
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.21.5</center>
</body>
</html>
```

\-> first use `curl -v` to get more info on the redirection :

```sh
controlplane $ curl world.universe.mine:30080/asia -v
*   Trying 172.30.1.2:30080...
* TCP_NODELAY set
* Connected to world.universe.mine (172.30.1.2) port 30080 (#0)
> GET /asia HTTP/1.1
> Host: world.universe.mine:30080
...
< HTTP/1.1 301 Moved Permanently
...
< Location: http://world.universe.mine/asia/		# <<<--- HERE
< 
<html>
<head><title>301 Moved Permanently</title></head>
...
```

\-> could be the **missing trailing slash**, use `-L` to follow the redirection together with `--connect-to ::world.universe.mine:30080` if not connecting to port 80 :

```sh
controlplane $ curl world.universe.mine:30080/asia -L --connect-to ::world.universe.mine:30080
hello, you reached ASIA
```

\-> could be the **path type** in the ingress rule (Prefix works better than Exact) :

- *Prefix* type :

  ```sh
  controlplane $ curl world.universe.mine:30080/asia -v
  ...
  < HTTP/1.1 301 Moved Permanently
  < Location: http://world.universe.mine/asia/
  ...
  
  controlplane $ curl world.universe.mine:30080/asia -L --connect-to ::world.universe.mine:30080
  hello, you reached ASIA
  ```

  \-> redirect working with `--connect-to`
- *Exact* type :

  ```sh
  controlplane $ curl world.universe.mine:30080/asia -v
  ...
  < HTTP/1.1 301 Moved Permanently
  < Location: http://world.universe.mine/asia/
  <head><title>301 Moved Permanently</title></head>
  ...
  
  controlplane $ curl world.universe.mine:30080/asia -L --connect-to ::world.universe.mine:30080
  <html>
  <head><title>404 Not Found</title></head>
  ...
  ```

  \-> redirect not working even with `--connect-to`

### Pods not deleting because of invalid tls certificate

- possible issue : tls certificate expired (unlikely if recent cluster) -> renew the certificates using kubeadm
- if certificate not yet valid and error displaying a date that is incorrect (e.g. 2020), check the date **on every machine in the cluster**

## Helm

possible d'utiliser des custom repositories

configurer helm pour utiliser le repo stable helm :

```bash
$ helm repo add stable https://charts.helm.sh/stable && helm repo update
```

créer une release test de wordpress (dispo dans le repo stable) :

```bash
$ helm install test stable/wordpress
```

structure minimale pour une chart :

- charts.yaml
- charts/
- templates/
- values.yaml

:warning: hook containers ne sont pas détruits automatiquement par un `helm delete` -> préciser la hook delete policy dans les annotations metadata



## EKS

- créer nodegroup (**avec** des instances EC2)

```bash
aws eks create-nodegroup --cluster-name eks-emma --nodegroup-name nodegroup-emma --subnets subnet-08f91c680d948f65b subnet-0f411b741cde8c050 --node-role arn:aws:iam::622746571813:role/EKSNodeRoleEmma --instance-types t2.micro
```

## Feedback formation

- cours cloudguru pas du tout suffisant pour la certif (pas toute la matière abordée, pas assez de détails et training trop facile)

## Divers

- `!!` en bash : remplacés par le texte de la commande précédente -> ex : `$ foo bar` puis `$ sudo !!` pour run la 1ère commande en sudo
- `$ foo bar && abc def` va run `abc def` après `foo bar` **seulement si la première commande return 0** -> `$ foo bar; abc def` va run la deuxième commande dans tous les cas
- `$ sudo apt-mark hold <package-name>` pour pas que le package s'update automatiquement

pbs de version api sur `kubectl` :  dans `~/.kube/config`, changer la version api de alpha à beta ([thread](https://gist.github.com/Zheaoli/335bba0ad0e49a214c61cbaaa1b20306))

downgrade kubectl :

```shell
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.23.6/bin/linux/amd64/kubectl
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

shell dans un pod :

```sh
$ kubectl exec --stdin --tty <pod-name> -- /bin/bash
# ou 
$ kubectl exec -it <pod_name> -- /bin/bash
```

logs applicatifs :

`cd /var/log` -> souvent dans `/var/log/messages`

get kernel release version : `uname -r`

troubleshooting pending pod :

- check le statut du pod en détails : ` kubectl describe po nginx-deployment-66b6c48dd5-bx8z2` -> si erreur du type `Too many pods`, possible de check le statut des nodes pour trouver la limite de pods/node avec `kubectl get nodes -o yaml | grep pods` -> avoir la liste des pods qui run sur un node : ` kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<node name here>`

troubleshooting crashloopbackoff :

- logs du pod qui a fail : `kubectl logs <pod_name> -p`
- infos sur le pod : ` kubectl describe po <pod_name>`