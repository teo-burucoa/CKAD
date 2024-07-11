# Exps exercices k8s

## Probes, init container, volume & env var

- create a pod running nginx with the following characteristics:
  - pod should not be marked ready if not responding correctly on a get request on nginx's web app root
  - startup should be considered completed only when the pod responds correctly on a get request on the nginx's web app root
  - pod should be restarted if the `ls` command inside the container does not work at some point
  - the content of the index.html file served by nginx should be replaced by its pod's name before startup of the actual nginx container

### Solution

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: probes
  name: probes
spec:
  replicas: 1
  selector:
    matchLabels:
      app: probes
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: probes
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
        ports:
        - containerPort: 80
          name: http
        volumeMounts:
        - name: nginx-data
          mountPath: /usr/share/nginx/
        
        # make sure the pod is ready by testing nginx's response on HTTP GET
        readinessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 2
          periodSeconds: 3
        # make sure the pod is alive by periodically running a ls command inside it
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - ls
        # wait for the nginx server to reploy correctly on a request on its web app root
        startupProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 2
          periodSeconds: 3
      
      # use an init container to edit nginx's index before starting nginx's container
      initContainers:
      - image: busybox
        name: init-box
        command:
        - sh
        - -c
        - 'mkdir -p "/usr/share/nginx/html/" && echo "Hello from $PODNAME" > /usr/share/nginx/html/index.html'
        env:
        # get the pod name using valueFrom.fieldRef
        - name: PODNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: nginx-data
          mountPath: /usr/share/nginx/
          
      # use a volume mount to enable the init container to write files used by the actual nginx container
      volumes:
      - name: nginx-data
        emptyDir: {}
```



## Jsonpath

Écrire une expression qui permet de récupérer les noms (space-separated) de tous les pods qui sont contrôlés par un ReplicaSet.
Déployer un pod (solo) et un déploiement de pods pour tester l'expression.

### Solution

```sh 
k get po -o jsonpath='{.items[?(@.metadata.ownerReferences[].kind=="ReplicaSet")].metadata.name} '
# pas oublier l'espace à la fin du jsonpath pour rendre le résultat plus lisible
```



## HostPath volume

Créer deux pods avec les capacités suivantes :

- un premier pod qui écrit (append) toutes les 10 secondes la date actuelle dans un fichier `/opt/shared/output` trouvable sur son hôte physique au même emplacement
- un second pod qui donne (sortie standard) toutes les 10 secondes le nombre de lignes présentes dans le fichier `/opt/shared/output` de son hôte physique



### Solution

- premier pod (writer.yaml) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: writer
  name: writer
spec:
  containers:
  - args:
    - sh
    - -c
    - while true; do date >> /opt/shared/output; sleep 10; done
    image: busybox
    name: writer
    resources: {}
    volumeMounts:
      - name: shared
        mountPath: /opt/shared
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: shared
    hostPath:
      path: /opt/shared
status: {}
```

- second pod (reader.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: reader
  name: reader
spec:
  containers:
  - args:
    - sh
    - -c
    - while true; do cat /opt/shared/output | wc -l; sleep 10; done
    image: busybox
    name: reader
    resources: {}
    volumeMounts:
      - name: shared
        mountPath: /opt/shared
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: shared
    hostPath:
      path: /opt/shared
status: {}
```


