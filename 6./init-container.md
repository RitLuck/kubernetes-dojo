#  Init Containers and Pod Lifecycle in Kubernetes

## Init Container

An **Init Container** is a special type of container that runs before the main application containers in a Pod start. It is primarily used to initialize the environment or perform setup tasks that must be completed prior to launching the main application.

- Runs before main containers: Init containers are executed before any container listed under `.spec.containers` in the Pod definition.
- Sequential execution: If multiple Init containers are defined, they are executed one at a time, in the order they appear in the manifest.
- Restart policy: Even if the Pod's restart policy is set to Always, Init containers will follow the `OnFailure` policy — meaning they will be retried if they fail.

### Why Use Init Containers?

Using an Init Container separately from your main application containers offers several advantages:

- Security isolation: You can include tools or scripts in the Init container that you don't want in the main application container, reducing attack surface.
- Modular design: Allows separation of responsibilities — for example, the Init container can handle setup logic, while the application container focuses on core functionality.
- Dependency management: If your application requires certain files, data, or external services to be available before starting, the Init container can handle these prerequisites.

##  Trying Out Init Containers

In the [5](/5./volume.md) exercise on Volumes, we created a Pod that writes logs to a file and another container in the same Pod that tails that file. However, because both containers start concurrently, there’s a chance the tail container might start before the file is created. If that happens, the tail container will fail since the file doesn’t exist yet — although Kubernetes will restart it automatically. (Which is pretty amazing!)

To address this timing issue, we can use an **Init Container** to create the log file before the main containers start. This ensures the file is present when the tail container begins.

Here's a manifest that uses an Init Container to initialize the log file before the date and tail containers run:

```
apiVersion: v1
kind: Pod
metadata:
  name: date-tail
spec:
  initContainers:
  - name: touch
    image: alpine
    command: ["sh", "-c"]
    args:
      - |
        echo "Initialize... (Started at $(date))"
        touch /var/log/date-tail/output.log
    volumeMounts:
      - name: log-volume
        mountPath: /var/log/date-tail

  containers:
  - name: date
    image: alpine
    command: ["sh", "-c"]
    args:
      - |
        echo "Append log... (Started at $(date))"
        exec >> /var/log/date-tail/output.log
        echo -n 'Start at: '
        while true; do
          date
          sleep 1
        done
    volumeMounts:
      - name: log-volume
        mountPath: /var/log/date-tail

  - name: tail
    image: alpine
    command: ["sh", "-c"]
    args:
      - |
        echo "Following... (Started at $(date))"
        tail -f /var/log/date-tail/output.log
    volumeMounts:
      - name: log-volume
        mountPath: /var/log/date-tail

  volumes:
    - name: log-volume
      emptyDir: {}

  terminationGracePeriodSeconds: 0
```


- The Init Container `touch` ensures the log file exists before the main containers start.
- The `date` container appends timestamps to the log file every second.
- The `tail` container continuously follows the log file and prints the output.


## Verify Init Container Execution Order

Create another Pod that has multiple Init Containers, and observe the order of execution.

```
apiVersion: v1
kind: Pod
metadata:
  name: initcontainer-test
spec:
  initContainers:
  - name: step1
    image: alpine
    command: ["sh", "-c"]
    args:
    - |
      echo Start at $(date); sleep 5; echo End.
  - name: step2
    image: alpine
    command: ["sh", "-c"]
    args:
    - |
      echo Start at $(date); sleep 5; echo End.
  - name: step3
    image: alpine
    command: ["sh", "-c"]
    args:
    - |
      echo Start at $(date); sleep 5; echo End.
  containers:
  - name: app1
    image: alpine
    command: ["sh", "-c"]
    args:
    - |
      echo Start at $(date); sleep 5; echo End.
  - name: app2
    image: alpine
    command: ["sh", "-c"]
    args:
    - |
      echo Start at $(date); sleep 5; echo End.
  restartPolicy: Never
  terminationGracePeriodSeconds: 0

```

### Run and Observe

```
kubectl apply -f initcontainer-test.yaml && kubectl get po -w                                  127 ✘  minikube ○  19:55:38 

pod/initcontainer-test created
NAME                 READY   STATUS     RESTARTS      AGE
date-tail            2/2     Running    2 (23s ago)   31h
initcontainer-test   0/2     Init:0/3   0             0s
initcontainer-test   0/2     Init:0/3   0             4s
initcontainer-test   0/2     Init:1/3   0             9s
initcontainer-test   0/2     Init:1/3   0             13s
initcontainer-test   0/2     Init:2/3   0             18s
initcontainer-test   0/2     Init:2/3   0             23s
initcontainer-test   0/2     PodInitializing   0             27s
initcontainer-test   2/2     Running           0             33s
initcontainer-test   1/2     NotReady          0             35s
initcontainer-test   0/2     Completed         0             38s
initcontainer-test   0/2     Completed         0             39s
^C%
```

### check each Init Container's logs:

```
kubectl logs initcontainer-test -c step1                                                  1 ✘  minikube ○  19:57:34 
Start at Wed Jul 23 15:55:47 UTC 2025
End.

kubectl logs initcontainer-test -c step2                                                    ✔  minikube ○  19:57:39 
Start at Wed Jul 23 15:55:55 UTC 2025
End.

kubectl logs initcontainer-test -c step3                                                    ✔  minikube ○  19:57:46 
Start at Wed Jul 23 15:56:05 UTC 2025
End.
```

### main containers:

```
kubectl logs initcontainer-test -c app1                                                   1 ✘  minikube ○  19:57:52 
Start at Wed Jul 23 15:56:12 UTC 2025
End.

kubectl logs initcontainer-test -c app2                                                   1 ✘  minikube ○  19:58:10 
Start at Wed Jul 23 15:56:16 UTC 2025
End.
```


## Pod Lifecycle

Each Pod goes through a phase, which represents its current state in the lifecycle. These phases give a high-level summary of what's happening with the Pod:

- Pending
The Pod has been accepted by the Kubernetes system, but one or more of its containers have not yet been created. This includes the time spent pulling container images and running any Init Containers

- Running
The Pod has been bound to a Node, and all containers have been created. At least one container is either running or is in the process of starting or restarting.

- Succeeded
All containers in the Pod have terminated successfully (i.e., exited with status 0) and will not be restarted.

- Failed
All containers in the Pod have terminated, but at least one container exited with a non-zero status (indicating failure) or was terminated by the system.

- Unknown
The Pod’s state cannot be determined, usually due to communication errors between the control plane and the node where the Pod is scheduled.