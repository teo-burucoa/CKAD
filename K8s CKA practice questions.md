# K8s CKA practice questions

## O'Reilly CKA

### **Networking**

#### Scenario 1 

Deploy a new deployment, **nginx**, with the latest image of **nginx** for two replicas in a namespace called **packt-app**. The container is exposed on port **80**. Create a service type of **ClusterIP** within the same namespace. Deploy a **sandbox-nginx** pod and make a call using **curl** to verify the connectivity to the **nginx** service. 

#### Scenario 2

Expose the **nginx** deployment with the **NodePort** service type; the container is exposed on port **80**. Use the **test-nginx** pod to make a call using **curl** to verify the connectivity to the **nginx** service. 

#### Scenario 3

Make a call using **wget** or **curl** from the machine within the same network as that node, to verify the connectivity with the **nginx** **NodePort** service through the correct port. 

#### Scenario 4

Use the **sandbox-nginx** pod and **nslookup** for the IP address of the **nginx** **NodePort** service. See what is returned. 

#### Scenario 5

Use the **sandbox-nginx** pod and **nslookup** for the DNS domain hostname of the **nginx** **NodePort** service. See what is returned. 

#### Scenario 6

Use the **sandbox-nginx** pod and **nslookup** for the DNS domain hostname of the **nginx** pod. See what is returned. 

You can find all the scenario resolutions in [*Appendix*](https://learning.oreilly.com/library/view/certified-kubernetes-administrator/9781803238265/B18201_Appendix_A.xhtml#_idTextAnchor386) *- Mock CKA scenario-based practice test resolutions* of this book.

### **App Scheduling & Lifecycle Management**

#### Scenario 1

SSH into the **worker-0** node and provision a new pod called **ngnix** with a single container, **nginx**.

#### Scenario 2

SSH to **worker-0** and then scale **nginx** to 5 copies. 

#### Scenario 3

SSH to **worker-0**, set a ConfigMap with a username and password, and then attach a new pod to BusyBox. 

#### Scenario 4

SSH to **worker-0** and create a **nginx** pod with an init container called **busybox**.

#### Scenario 5

SSH to **worker-0**, create a **nginx** pod, and then a **busybox** container in the same pod.



### K8s Storage

#### Scenario 1

Create a new PV called **packt-data-pv** to store 2 GB, and two PVCs each requesting 1 GB of local storage. 

#### Scenario 2

Provision a new pod called **pack-storage-pod** and assign an available PV to this Pod.

You can find all the scenario resolutions in [*Appendix*](https://learning.oreilly.com/library/view/certified-kubernetes-administrator/9781803238265/B18201_Appendix_A.xhtml#_idTextAnchor386) *- Mock CKA scenario-based practice test resolutions* of this book.



### Security

#### Scenario 1 

Create a new service account named **packt-sa** in a new namespace called **packt-ns**. 

#### Scenario 2

Create a Role named **packtrole** and bind it with the RoleBinding **packt-clusterbinding**. Map the **packt-sa** service account with **list** and **get** permissions.

#### Scenario 3 

Create a new pod named **packt-pod** with the **busybox:1.28** image in the **packt-ns** namespace. Expose port **80**. Then, assign the **packt-sa** service account to the Pod. 

You can find all the scenario resolutions in [*Appendix*](https://learning.oreilly.com/library/view/certified-kubernetes-administrator/9781803238265/B18201_Appendix_A.xhtml#_idTextAnchor386) *- Mock CKA scenario-based practice test resolutions* of this book.