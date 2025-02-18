![](https://gaforgithub.azurewebsites.net/api?repo=CKAD-exercises/observability&empty)
## Observability (18%)
### Practice questions based on these concepts

* Understand LivenessProbes and ReadinessProbes
* Understand Container Logging
* Understand how to monitor applications in kubernetes
* Understand Debugging in Kubernetes


## Liveness and readiness probes

kubernetes.io > Documentation > Tasks > Configure Pods and Containers > [Configure Liveness and Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)

### Create an nginx pod with a liveness probe that just runs the command 'ls'. Save its YAML in pod.yaml. Run it, check its probe status, delete it.

<details><summary>show</summary>
<p>

```bash
kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > pod.yaml
vi pod.yaml
```

```YAML
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    livenessProbe: # our probe
      exec: # add this line
        command: # command definition
        - ls # ls command
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```bash
kubectl create -f pod.yaml
kubectl describe pod nginx | grep -i liveness # run this to see that liveness probe works
kubectl delete -f pod.yaml
```

</p>
</details>

### Modify the pod.yaml file so that liveness probe starts kicking in after 5 seconds whereas the interval between probes would be 5 seconds. Run it, check the probe, delete it.

<details><summary>show</summary>
<p>

```bash
kubectl explain pod.spec.containers.livenessProbe # get the exact names
```

```YAML
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    livenessProbe: 
      initialDelaySeconds: 5 # add this line
      periodSeconds: 5 # add this line as well
      exec:
        command:
        - ls
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```bash
kubectl create -f pod.yaml
kubectl describe po nginx | grep -i liveness
kubectl delete -f pod.yaml
```

</p>
</details>

### Create an nginx pod (that includes port 80) with an HTTP readinessProbe on path '/' on port 80. Again, run it, check the readinessProbe, delete it.

<details><summary>show</summary>
<p>

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml --restart=Never --port=80 > pod.yaml
vi pod.yaml
```

```YAML
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    ports:
      - containerPort: 80 # Note: Readiness probes runs on the container during its whole lifecycle. Since nginx exposes 80, containerPort: 80 is not required for readiness to work.
    readinessProbe: # declare the readiness probe
      httpGet: # add this line
        path: / #
        port: 80 #
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

```bash
kubectl create -f pod.yaml
kubectl describe pod nginx | grep -i readiness # to see the pod readiness details
kubectl delete -f pod.yaml
```

</p>
</details>

### Lots of pods are running in `qa`,`alan`,`test`,`production` namespaces.  All of these pods are configured with liveness probe.  Please list all pods whose liveness probe are failed in the format of `<namespace>/<pod name>` per line.

<details><summary>show</summary>
<p>

A typical liveness probe failure event
```
LAST SEEN   TYPE      REASON      OBJECT              MESSAGE
22m         Warning   Unhealthy   pod/liveness-exec   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

collect failed pods namespace by namespace

```sh  
kubectl get ns # check namespaces
kubectl -n qa get events | grep -i "Liveness probe failed"
kubectl -n alan get events | grep -i "Liveness probe failed"
kubectl -n test get events | grep -i "Liveness probe failed"
kubectl -n production get events | grep -i "Liveness probe failed"
```

</p>
</details>

## Logging

### Create a busybox pod that runs 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done'. Check its logs

<details><summary>show</summary>
<p>

```bash
kubectl run busybox --image=busybox --restart=Never -- /bin/sh -c 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done'
kubectl logs busybox -f # follow the logs
```

</p>
</details>

## Debugging

### Create a busybox pod that runs 'ls /notexist'. Determine if there's an error (of course there is), see it. In the end, delete the pod

<details><summary>show</summary>
<p>

```bash
kubectl run busybox --restart=Never --image=busybox -- /bin/sh -c 'ls /notexist'
# show that there's an error
kubectl logs busybox
kubectl describe po busybox
kubectl delete po busybox
```

</p>
</details>

### Create a busybox pod that runs 'notexist'. Determine if there's an error (of course there is), see it. In the end, delete the pod forcefully with a 0 grace period

<details><summary>show</summary>
<p>

```bash
kubectl run busybox --restart=Never --image=busybox -- notexist
kubectl logs busybox # will bring nothing! container never started
kubectl describe po busybox # in the events section, you'll see the error
# also...
kubectl get events | grep -i error # you'll see the error here as well
kubectl delete po busybox --force --grace-period=0
```

</p>
</details>


### Get CPU/memory utilization for nodes ([metrics-server](https://github.com/kubernetes-incubator/metrics-server) must be running)

<details><summary>show</summary>
<p>

```bash
kubectl top nodes
```

</p>
</details>

### Additonal Practice Questions

<details><summary>Create an nginx pod with containerPort 80 and it should only receive traffic only it checks the endpoint / on port 80 and verify and delete the pod</summary>
<p>

```
kubectl run nginx --image=nginx --restart=Never --port=80 --dry-run -o yaml > nginx-pod.yaml

// add the readinessProbe section and create
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

kubectl create -f nginx-pod.yaml

// verify
kubectl describe pod nginx | grep -i readiness
kubectl delete po nginx
```
</p>
</details>


<details><summary>Create an nginx pod with containerPort 80 and it should check the pod running at endpoint / healthz on port 80 and verify and delete the pod</summary>
<p>

```
kubectl run nginx --image=nginx --restart=Never --port=80 --dry-run -o yaml > nginx-pod.yaml

// add the livenessProbe section and create
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

kubectl create -f nginx-pod.yaml

// verify
kubectl describe pod nginx | grep -i readiness
kubectl delete po nginx
```
</p>
</details>


<details><summary>Create an nginx pod with containerPort 80 and it should check the pod running at endpoint /healthz on port 80 and it should only receive traffic only it checks the endpoint / on port 80. verify the pod</summary>
<p>

```
kubectl run nginx --image=nginx --restart=Never --port=80 --dry-run -o yaml > nginx-pod.yaml

// add the livenessProbe and readiness section and create
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /healthz
        port: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

kubectl create -f nginx-pod.yaml

// verify
kubectl describe pod nginx | grep -i readiness
kubectl describe pod nginx | grep -i liveness
```
</p>
</details>


<details><summary>Check what all are the options that we can configure with readiness and liveness probes</summary>
<p>

```
kubectl explain Pod.spec.containers.livenessProbe
kubectl explain Pod.spec.containers.readinessProbe
```
</p>
</details>


<details><summary>Create the pod nginx with the above liveness and readiness probes so that it should wait for 20 seconds before it checks liveness and readiness probes and it should check every 25 seconds.</summary>
<p>

```
// nginx-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      initialDelaySeconds: 20
      periodSeconds: 25
      httpGet:
        path: /healthz
        port: 80
    readinessProbe:
      initialDelaySeconds: 20
      periodSeconds: 25
      httpGet:
        path: /
        port: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

kubectl create -f nginx-pod.yaml
```
</p>
</details>


<details><summary>Create a busybox pod with this command “echo I am from busybox pod; sleep 3600;” and verify the logs</summary>
<p>

```
kubectl run busybox --image=busybox --restart=Never -- /bin/sh -c "echo I am from busybox pod; sleep 3600;"

kubectl logs busybox
```
</p>
</details>


<details><summary>copy the logs of the above pod to the busybox-logs.txt and verify</summary>
<p>

```
kubectl logs busybox > busybox-logs.txt

cat busybox-logs.txt
```
</p>
</details>


<details><summary>List all the events sorted by timestamp and put them into file.log and verify</summary>
<p>

```
kubectl get events --sort-by=.metadata.creationTimestamp

// putting them into file.log
kubectl get events --sort-by=.metadata.creationTimestamp > file.log

cat file.log
```
</p>
</details>


<details><summary>Create a pod with an image alpine which executes this command ”while true; do echo ‘Hi I am from alpine’; sleep 5; done” and verify and follow the logs of the pod</summary>
<p>

```
// create the pod
kubectl run hello --image=alpine --restart=Never  -- /bin/sh -c "while true; do echo 'Hi I am from Alpine'; sleep 5;done"

// verify and follow the logs
kubectl logs --follow hello
```
</p>
</details>


<details><summary>Create the pod with this kubectl create -f https://gist.githubusercontent.com/bbachi/212168375b39e36e2e2984c097167b00/raw/1fd63509c3ae3a3d3da844640fb4cca744543c1c/not-running.yml. The pod is not in the running state. Debug it.</summary>
<p>

```
// create the pod
kubectl create -f https://gist.githubusercontent.com/bbachi/212168375b39e36e2e2984c097167b00/raw/1fd63509c3ae3a3d3da844640fb4cca744543c1c/not-running.yml

// get the pod
kubectl get pod not-running
kubectl describe po not-running

// it clearly says ImagePullBackOff something wrong with image
kubectl edit pod not-running // it will open vim editor
                     or
kubectl set image pod/not-running not-running=nginx
```
</p>
</details>


<details><summary>This following yaml creates 4 namespaces and 4 pods. One of the pod in one of the namespaces are not in the running state. Debug and fix it. `https://gist.githubusercontent.com/bbachi/1f001f10337234d46806929d12245397/raw/84b7295fb077f15de979fec5b3f7a13fc69c6d83/problem-pod.yaml`</summary>
<p>

```
kubectl create -f https://gist.githubusercontent.com/bbachi/1f001f10337234d46806929d12245397/raw/84b7295fb077f15de979fec5b3f7a13fc69c6d83/problem-pod.yaml

// get all the pods in all namespaces
kubectl get po --all-namespaces

// find out which pod is not running
kubectl get po -n namespace2

// update the image
kubectl set image pod/pod2 pod2=nginx -n namespace2

// verify again
kubectl get po -n namespace2
```
</p>
</details>


<details><summary>Get the memory and CPU usage of all the pods and find out top 3 pods which have the highest usage and put them into the cpu-usage.txt file</summary>
<p>

```
// get the top 3 hungry pods
kubectl top pod --all-namespaces | sort --reverse --key 3 --numeric | head -3

// putting into file
kubectl top pod --all-namespaces | sort --reverse --key 3 --numeric | head -3 > cpu-usage.txt

// verify
cat cpu-usage.txt
```
</p>
</details>

## Defining a Pod’s Readiness and Liveness Probe

1. Create a new Pod named `hello` with the image `bonomat/nodejs-hello-world` that exposes the port 3000. Provide the name `nodejs-port` for the container port.
2. Add a Readiness Probe that checks the URL path / on the port referenced with the name `nodejs-port` after a 2 seconds delay. You do not have to define the period interval.
3. Add a Liveness Probe that verifies that the app is up and running every 8 seconds by checking the URL path / on the port referenced with the name `nodejs-port`. The probe should start with a 5 seconds delay.
4. Shell into container and curl `localhost:3000`. Write down the output. Exit the container.
5. Retrieve the logs from the container. Write down the output.

<details><summary>Show Solution</summary>
<p>

Create the intial YAML with the following command.

```bash
$ kubectl run hello --image=bonomat/nodejs-hello-world --restart=Never --port=3000 -o yaml --dry-run > pod.yaml
```

Edit the YAML file and add the probes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: hello
  name: hello
spec:
  containers:
  - image: bonomat/nodejs-hello-world
    name: hello
    ports:
    - name: nodejs-port
      containerPort: 3000
    readinessProbe:
      httpGet:
        path: /
        port: nodejs-port
      initialDelaySeconds: 2
    livenessProbe:
      httpGet:
        path: /
        port: nodejs-port
      initialDelaySeconds: 5
      periodSeconds: 8
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Create the Pod from the YAML file, shell into the Pod as soon as it is running and execute the `curl` command.

```bash
$ kubectl create -f pod.yaml
pod/hello created
$ kubectl exec hello -it -- /bin/sh
/ # curl localhost:3000
<!DOCTYPE html>
<html>
<head>
	<title>NodeJS Docker Hello World</title>
	<meta charset="utf-8">
	<meta name="viewport" content="width=device-width, initial-scale=1">
	<link href="http://cdn.bootcss.com/bootstrap/3.3.2/css/bootstrap.min.css" rel="stylesheet">
	<link href="/stylesheets/styles.css" rel="stylesheet">
</head>
<body>
	<div class="container">
		<div class="well well-sm">
			<h2>This is just a hello world message</h2>
			<img a href="./cage.jpg"/>
			<img src="src/cage.jpg" alt="Smiley face" width="640">
		</div>
	</div>
</body>
</html>
/ # exit

$ kubectl logs pod/hello
Magic happens on port 3000
```

</p>
</details>

## Fixing a Misconfigured Pod

1. Create a new Pod with the following YAML.

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: failing-pod
  name: failing-pod
spec:
  containers:
  - args:
    - /bin/sh
    - -c
    - while true; do echo $(date) >> ~/tmp/curr-date.txt; sleep
      5; done;
    image: busybox
    name: failing-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

2. Check the Pod's status. Do you see any issue?
3. Follow the logs of the running container and identify an issue.
4. Fix the issue by shelling into the container. After resolving the issue the current date should be written to a file. Render the output.

<details><summary>Show Solution</summary>
<p>

First, create the Pod with the given YAML content.

```bash
$ vim pod.yaml
$ kubectl create -f pod.yaml
```

The Pod seems to be running without problems.

```bash
$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
failing-pod   1/1     Running   0          5s
```

Render the logs of the container. The output should indicate an error message every 5 seconds.

```bash
$ kubectl logs failing-pod
Unable to write file!
/bin/sh: 1: cannot create /root/tmp/curr-date.txt: Directory nonexistent
Unable to write file!
/bin/sh: 1: cannot create /root/tmp/curr-date.txt: Directory nonexistent
Unable to write file!
/bin/sh: 1: cannot create /root/tmp/curr-date.txt: Directory nonexistent
```

Apparently, the directory we want to write to does not exist. Log into the container and create the directory. The file `~/tmp/curr-date.txt` is populated.

```bash
$ kubectl exec failing-pod -it -- /bin/sh
/ # mkdir -p ~/tmp
/ # cd ~/tmp
/ # ls -l
total 4
-rw-r--r-- 1 root root 112 May  9 23:52 curr-date.txt
/ # cat ~/tmp/curr-date.txt
Thu May 9 23:59:01 UTC 2019
Thu May 9 23:59:06 UTC 2019
Thu May 9 23:59:11 UTC 2019
/ # exit
```

</p>
</details>
