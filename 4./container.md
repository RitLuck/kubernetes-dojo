# The Container Object field.

### Command

- Specify the array of the Entrypoint. In short, specify the process that you want to start in the container.

`command: ["sh", "-c"]`


### args

```
args: |
  echo hello world from $HOSTNAME
  date
  sleep 3
```

### env

```
env:
  HOGEHOGE: fugafuga
  FOO: bar
```

### The image

```
image: nginx:alpine
```


### imagePullPolicy

Specify a policy for how Kubernetes pulls container images. 

- `Always`: Always pull the container image, even if it already exists locally.
- `IfNotPresent`: Do not run Pull if there is already a container image
- `Never`: Do not run the pull. Expect to have a container image locally

If you don’t set an image pull policy, Kubernetes decides based on the image tag:

- If the image tag is `latest`, Kubernetes defaults to `Always`.
- If the image tag is anything other than `latest`, it defaults to `IfNotPresent`.

```
imagePullPolicy: Always
```

### name

Specify the name of the container. The name must follow the [DNS_LABEL](https://datatracker.ietf.org/doc/html/rfc1035#page-8) format:

- It must start with a letter (a–z).
- It can contain letters, digits (0–9), and hyphens (-) in the middle.
- It must end with a letter or digit.
- The name must be unique within the Pod.
- This field cannot be updated after the container is created.

```
name: nginx-pod
```


### Ports

You can specify which port(s) should be exposed from the container. This is done using the ports field.

    ⚠️ This field cannot be updated after the container is created.


- `containerPort`:  The port number to expose from the container. It will be accessible via the Pod’s IP.
- `name`: A name for the port, used as a Named Port (e.g., for network policies). Must be unique within the Pod.
- `protocol`:The protocol to use. Valid values are TCP, UDP, or SCTP. Defaults to TCP if not specified.

```
ports:
  - name: http
    containerPort: 8080
```

### working Dir

specify the working directory for the container using the `workingDir` field.

- if omitted, the container runtime will use its default working directory (usually `/`).
- If a working directory is defined inside the container image, that may be used instead.

```
workingDir: /tmp
```


### Example

Let's create a Pod using the `nginx:alpine` image and explicitly set fields like `imagePullPolicy`, `command`, `args`, `env`, and `workingDir`.


```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    imagePullPolicy: Always
    command: []
    args: ["nginx", "-g", "daemon off;"]
    env:
    - name: HOGEHOGE
      value: fugafuga
    ports:
    - containerPort: 80
      protocol: TCP
    workingDir: /tmp

```

- Save the file as nginx.yaml and apply it:

```
kubectl apply -f nginx.yaml                                                                                              
pod/nginx created
```

- Verify the Pod is running:

```
kubectl get pods                                                                                                          ✔  minikube ○  13:03:59 

NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          4s

```

- Check the Running Process.Use `kubectl exec` to inspect the process inside the container:

```
kubectl exec nginx -- ps                                                                                                  ✔  minikube ○  13:04:02 
PID   USER     TIME  COMMAND
    1 root      0:00 nginx: master process nginx -g daemon off;
   31 nginx     0:00 nginx: worker process
   32 nginx     0:00 nginx: worker process
   33 root      0:00 ps
```


###  Check Environment Variables

Let's see if the `HOGEHOGE` environment variable was set correctly:

```
kubectl exec nginx -- env                                                                                               1 ✘  minikube ○  13:05:39 
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx
HOGEHOGE=fugafuga
CV_SERVICE_PORT_80_TCP_ADDR=10.107.51.30
```

 HOGEHOGE=fugafuga is present, confirming the env variable was applied.


To confirm the workingDir is `/tmp`, we can inspect the symbolic link to the process’s current directory:

```
kubectl exec nginx -- readlink /proc/1/cwd
/tmp
```

As specified, the working directory is correctly set to /tmp.