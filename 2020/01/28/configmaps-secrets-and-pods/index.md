# ConfigMaps, Secrets and Pods

> A Pod is collection of containers running is shared namespace. A container is created from Docker/OCI Image, which is immutable in nature i.e. data bundled into image cannot be changed later in image. “While immutable image of application is great for security, but putting configurations & data like security keys is bad for security as well as portability” There are various ways to pass data to application in containers, without changing the flow of application e.

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2020-01-28
- Tags: Kubernetes, ConfigMaps, Secrets
- Canonical: https://kunalkushwaha.github.io/2020/01/28/configmaps-secrets-and-pods/

A Pod is collection of containers running is shared namespace. A container is created from Docker/OCI Image, which is immutable in nature i.e. data bundled into image cannot be changed later in image.

> “While immutable image of application is great for security, but putting configurations & data like security keys is bad for security as well as portability”

There are various ways to pass data to application in containers, without changing the flow of application e.g. using setting environment variables or use volume mount to provide configuration file at container creation.

In Kubernetes, ConfigMaps and Secrets are two objects using which configuration data can be passed to Pods and containers running within them.

*In this blog, I will give you a walkthough of how to use ConfigMap and Secrets with a Pod.*

---

**ConfigMaps:** “It allow you to decouple configuration artifacts from image content to keep containerized applications portable”

- It stores the data in key-value pair, where value can be a single word to data of a file.

### Creating ConfigMap

- Since ConfigMap is standard object of Kubernetes, it can be created using command `kubectl create configmap` as well as `kubectl apply` with ConfigMap definition file.

e.g. ConfigMap with two key-value pairs `log=debug` & `environment=test`

```
$ kubectl create configmap testapp-pod-config --from-literal=log=debug --from-literal=environment=test
configmap/testapp-pod-config created
```

Literal values can be seen using with `kubectl describe` command

```
$ kubectl describe configmap testapp-pod-config
Name:         testapp-pod-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
environment:
----
test
log:
----
debug
Events:  <none>
```

Simillarly, The ConfigMap can be created using files too.

e.g. creating a config map to store ha-proxy config stored in `./ha-proxy.cfg` file

```
$ kubectl create configmap ha-proxy-cfg --from-file=ha-proxy-cfg=ha-proxy-config.cfg
configmap/ha-proxy-cfg created
```

You may go through various examples of ConfigMap creation in [Kubernetes documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-a-configmap)

### How to use it with Pod?

ConfigMap data can be consumed in Pod broadly either as environment variable or mount them as files in container.
Lets see in detail, how this is done.

#### 1. Import all data as environment variable:

This is simplest form, where we will import all data present in a ConfigMap into the Pod as environment variable.
Use `envFrom` & `configMapRef` to import all. I the below example, I am importing all key-value pair of ConfigMap`testapp-pod-config` created above.

```
apiVersion: v1
kind: Pod
metadata:
    name: testapp-pod
spec:
    containers:
    - name: testapp-container
    image: docker.io/library/busybox:latest
    command: ["/bin/sh","-c","env"]
    envFrom:
    - configMapRef:
            name: testapp-pod-config
```

You can see below, both key-value pair are now part of environment variable of Pod.

```
$ kubectl apply -f testapp-pod.yaml
pod/testapp-pod created

$ kubectl logs testapp-pod | grep -E -i -w  'log|environment'
environment=test
log=debug
```

---

**Info**

*If you are looking at container environment variable first time, don’t get intimidated with list of enviornment list not related to your pod/container.*

*A list of all services that were running when a Container is created is available to that Container as [environment variables.](https://kubernetes.io/docs/concepts/containers/container-environment-variables/)*

---

#### 2. Import selected keys as environment variable.

Sometime, not all key-value pairs are required to be exposed to container especially when same ConfigMap is used by multiple Pods. In such cases, we can import selected keys to a container.

This can be done using `env` , `valueFrom` & `configMapKeyRef`.
Below is example, where we import only `log` key from `test-pod-config`

```
apiVersion: v1
kind: Pod
metadata:
    name: testapp-pod-single-key
spec:
    containers:
    - name: testapp-container
      image: docker.io/library/busybox:latest
      command: ["/bin/sh","-c","env"]
      env:
      - name: "log-level"
        valueFrom:
          configMapKeyRef:
            name: testapp-pod-config
            key: "log" 
```

You can see only `log` key is imported and its renamed as `log-level` as defined in above yaml file.
Hence while importing key wise, we can even rename the environment varaibles too.

```
$ kubectl apply -f example-single-config-pod.yaml 
pod/testapp-pod-single-key created

$ kubectl logs testapp-pod-single-key | grep -E -i -w  'log|environment'
log-level=debug
```

---

**Info**

*Since, `env` and `envFrom` are defined as list, environment list from multiple ConfigMaps can be imported.*

---

#### 3. Import as volume.

This option is helpful, if application bundled in container which needs to be configured using config file.
We can mount the file to specific path within container.

For this, we need to first use ConfigMap as volume of Pod and then mount within the container.

In below example, a pod will mount the ConfigMap `ha-proxy-cfg`.

```
apiVersion: v1
kind: Pod
metadata:
    name: testapp-pod-mnt
spec:
    volumes:
    - name: ha-proxy-volume
      configMap:
        name: ha-proxy-cfg
    containers:
    - name: testapp-container
      image: docker.io/library/busybox:latest
      command: ["/bin/sh","-c","ls -l /etc/config/"]
      volumeMounts:
      - name: ha-proxy-volume
        mountPath: "/etc/config"
```

You can see below, the key became filename and value assigned to key will be contents of file(*not shown*)
Using `configMap` & just `name` items, all the keys defined in ConfigMap will be mounted on provided `mountPath`

```
$ kubectl get configmaps
NAME                 DATA   AGE
ha-proxy-cfg         1      20h

$ kubectl apply -f mount-file-example.yaml                                      
pod/testapp-pod-mnt created

$ kubectl logs testapp-pod-mnt
total 0
lrwxrwxrwx    1 root     root            19 Jan 29 05:17 ha-proxy-cfg -> ..data/ha-proxy-cfg
```

#### 4. Import selected keys to volume.

Again similar to #2 case, selected keys can be imported within mounted volume.
This can be done by putting keys in `volumes` section of Pod as shown below.

```
apiVersion: v1
kind: Pod
metadata:
    name: testapp-pod-mnt-single
spec:
    volumes:
    - name: ha-proxy-volume-single
      configMap:
        name: ha-proxy
        items:
        - key: "ha-proxy-cfg"
          path: ha-proxy.cfg
    containers:
    - name: testapp-container
      image: docker.io/library/busybox:latest
      command: ["/bin/sh","-c","ls -l /etc/config/"]
      volumeMounts:
      - name: ha-proxy-volume-single
        mountPath: /etc/config
```

As you can see, the filename can be chosen in this case.

```
$ kubectl apply -f mount-single-file-example.yaml
pod/testapp-pod-mnt-single created

$ kubectl logs testapp-pod-mnt-single
total 0
lrwxrwxrwx    1 root     root            19 Jan 29 05:48 ha-proxy.cfg -> ..data/ha-proxy.cfg
```

---

**Info**

*Any changes to ConfigMap will be refletcted in volumes too.
So this is helpfull in cases where application bundled in containers can read changes and you dont need to create new pods to propogate the changes in configuration files*

---

### Secret:
