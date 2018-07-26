## Pod
As per the [Kubernetes Documents](https://kubernetes.io/docs/concepts/workloads/pods/pod/), “Pods are the smallest deployable units of computing that can be created and managed in Kubernetes”. Pod can contain one or more containers. Pod can be created with configuration file or with Command line.

Create pod from following `pod-nginx.yaml` file.
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
     app: nginx
spec:
  containers:
  - name: nginx-demo
    image: nginx:1.9.1
    ports:
    - containerPort: 80
```
Deploy the pod from above yaml file.
```
$ kubectl create -f pod-nginx.yaml
pod "nginx" created
```
 Get list of running Pods.
```
$ kubectl get pod
NAME      READY     STATUS              RESTARTS   AGE
nginx     0/1       ContainerCreating   0          42s
```

## Multi containers  Pod
As we have seen the pod can consiste of multiple containers. We are going to demonstrate pod with multipe containers.

Create Pod with Multi containers from following yaml file.
```
apiVersion: v1
kind: Pod
metadata:
  name: multicontainer
spec:
  volumes:
  - name: html
    emptyDir: {}
  containers:
  - name: con1
    image: nginx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  - name: con2
    image: debian
    volumeMounts:
    - name: html
      mountPath: /html
    command: ["/bin/sh", "-c"]
    args:
      - while true; do
          date >> /html/index.html;
          sleep 1;
        done
```
In above cofiguration file we have specified two containers `con1` and `con2`. Both containers are running and in the single pod and sharing the single volume.

Create pod from above yaml file.
```
$ kubectl create -f multicontainer.yaml
pod "multicontainer" created
```
Check the list of Pods.
```
$ kubectl get po
NAME             READY     STATUS              RESTARTS   AGE
multicontainer   0/2       ContainerCreating   0          2m
```

## Init container Pod

Kubernetes have included an alpha feature called as init containers.Init containers are one or more containers in a pod that get run and complete its execution prior to the other application containers starting. An init container is exactly like a regular container, except that it always runs to completion and Execution of each init container must be  successfully completed before the next container is started.

Create Pod which contains init container from following yaml.
```
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
    - image: nginx:1.9.1
      name: cloudyuga-nginx
      volumeMounts:
      - name: demo
        mountPath: /tmp/
  initContainers:
    - image: nginx:1.9.1
      name: demo
      command: ["/bin/sh"]
      args: ["-c", "echo cloudyuga >>/tmp/demo.txt"]
      volumeMounts:
      - name: demo
        mountPath: /tmp/
  restartPolicy: Never
  volumes:
    - name: demo
      hostPath:
        path: /tmp

```
For simplycity we have created a pod in which init container runs and write `cloudyuga`in directory which is mounted with volume.
When the init container run and successfully completes its execution then another container `cloudyuga-nginx` get running and the same volume is also mounted within this container. hence the file wriiten by the init container is also mounted in this container also.

Deploy above yaml file.
```
$ kubectl create -f init-pod.yaml
pod "init-demo" created
```
Check the list of Pods.
```
kubectl get po
NAME        READY     STATUS    RESTARTS   AGE
init-demo   1/1       Running   0          7s
```
Lets check the File created by the init container is present. For that ssh inti the pod n check the /tmp/demo.txt file.
```
$ kubectl exec -it init-demo sh
# cat /tmp/demo.txt
cloudyuga
#
```
the `demo.txt` created by init-container was in `demo` volume and the same volume is mounted in `cloudyuga-nginx` containers.
Init containers gets executed before the `cloudyuga-nginx` containers.

## Pod With Health Check.
A liveness probe we used in following demo, determines whether the container is running or not. If the liveness probe fails, then the container get killed. New container can be started as per mentioned in restart policy .

Create pod from following yaml file. This pod is correctly configured with liveness probe on 80 port.
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-liveness
  labels:
     app: nginx
spec:
  containers:
  - name: nginx-demo
    image: nginx:1.9.1
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 5
      timeoutSeconds: 1
```
In above configuration we have declared that after 15 seconds,  Kubernetes start checking the health/ endpoint every 5 seconds.

Deploy above pod.
```
$ kubectl create -f nginx-liveness.yaml
pod "nginx-liveness" created
```
Get the running pod list. This pod is correctly configured with liveness probe on 80 port. There may be no restarts.
```
$ kubectl get po
NAME             READY     STATUS    RESTARTS   AGE
nginx-liveness   1/1       Running   0          8s
```

Now lets create pod from following yaml file. This pod is not correctly configured with liveness probe on 81 port.
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-liveness-fail
  labels:
     app: nginx
spec:
  containers:
  - name: nginx-demo
    image: nginx:1.9.1
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 81
      initialDelaySeconds: 15
      timeoutSeconds: 1
```
Deploy above pod.
```
$ kubectl create -f nginx-liveness-fail.yaml
pod "nginx-liveness-fail" created
```
Get the list of running pods. This pod is not correctly configured with liveness probe on 81 port.
```
$ kubectl get po
NAME                  READY     STATUS    RESTARTS   AGE
nginx-liveness        1/1       Running   0          8m
nginx-liveness-fail   1/1       Running   4          2m

```
You can get Addition information about this by executing this command `kubectl describe po nginx-liveness-fail`
