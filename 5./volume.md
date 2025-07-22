# Volume

When a container runs in a Pod, any data stored inside it is temporary. If the Pod or container is deleted, or if the container crashes and Kubernetes restarts it, the data will be lost.

To persist data beyond the lifecycle of a container, Kubernetes provides a mechanism called **Volumes**. You can store important data in a Volume, which exists independently of the container. This means the data remains intact even if the container is deleted or crashes.

Volumes also serve another important purpose: 

- data sharing between containers within the same Pod. 
Since containers in a Pod can access the same Volume, they can read and write to shared data, enabling smooth communication or cooperation.

```
+--------------------+
|       Pod          |
|  +-------------+   |
|  | Container A |   |
|  +-------------+   |
|         |          |
|     Shared Volume  |
|         |          |
|  +-------------+   |
|  | Container B |   |
|  +-------------+   |
+--------------------+
```

There are several types of Volumes in Kubernetes, including:

- emptyDir
- hostPath
- configMap
- secret
- gcePersistentDisk
- awsElasticBlockStore
- csi
- downwardAPI
- nfs

...and others.

Some of these, like **`awsElasticBlockStore`** and **`gcePersistentDisk`**, are used when running Kubernetes on cloud platforms such as AWS or Google Cloud. They allow you to create and mount volumes backed by cloud storage services.

However, since we are using a local Kubernetes environment with Minikube, we can only use volume types that work in local setups, such as `emptyDir` and `hostPath`.


## `emptyDir`

This type of volume is temporary and exists only as long as the Pod is running. It's ideal for sharing data between containers within the same Pod.

```
apiVersion: v1
kind: Pod
metadata:
  name: date-tail
spec:
  containers:
  - name: date
    image: alpine
    command: ["sh", "-c"]
    args:
    - |
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
      tail -f /var/log/date-tail/output.log
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/date-tail
  volumes:
  - name: log-volume
    emptyDir: {}
  terminationGracePeriodSeconds: 0
```

This configuration defines:

- A volume named `log-volume` using `emptyDir`.
- Two containers:   
  - date: writes the current date and time every second to output.log.
  - tail: tails the same output.log file.
- Both containers mount the volume at /var/log/date-tail.


### Deploy the Pod

```
kubectl apply -f date-tail.yaml                                                              1 ✘  minikube ○  11:46:56 
pod/date-tail created
```

### Verify the Shared Volume

```
kubectl exec -it date-tail -c date -- ls /var/log/date-tail                                 ✔  minikube ○  11:47:12 
output.log

kubectl exec -it date-tail -c tail -- ls /var/log/date-tail                                 ✔  minikube ○  11:47:36 
output.log
```

You can see that both containers can access the same file.


### View Logs

```
kubectl logs date-tail -c tail -f --tail=10                                                 ✔  minikube ○  11:47:46 
Tue Jul 22 07:47:47 UTC 2025
Tue Jul 22 07:47:48 UTC 2025
Tue Jul 22 07:47:49 UTC 2025
Tue Jul 22 07:47:50 UTC 2025
...
```

The output shows that the file is being updated in real-time and shared between containers.


### What Happens After Pod Deletion?

Try force-replacing the Pod:

```
kubectl replace --force -f date-tail.yaml                                                   1 ✘  5s  minikube ○  11:48:02 
pod "date-tail" deleted
pod/date-tail replaced
```

Check the logs again:

```
kubectl logs date-tail -c tail -f                                                         1 ✘  minikube ○  11:51:34 
Start at: Tue Jul 22 07:51:33 UTC 2025
Tue Jul 22 07:51:34 UTC 2025
Tue Jul 22 07:51:35 UTC 2025
Tue Jul 22 07:51:36 UTC 2025
Tue Jul 22 07:51:37 UTC 2025
Tue Jul 22 07:51:38 UTC 2025
...
```

The `Start at` timestamp has reset. This means the file was recreated, and previous data was lost. That’s because `emptyDir` volumes are not persistent,they are deleted when the Pod is deleted.

## Cleanup

```
kubectl delete -f date-tail.yaml                                                          1 ✘  minikube ○  11:51:40 
pod "date-tail" deleted
```

## `hostPath`

In the previous example, we used an `emptyDir` volume to share data between containers in a Pod. However, `emptyDir` is temporary and is deleted when the Pod is removed. If you want the data to **persist** even after recreating the Pod,at least locally in Minikube,you can use a `hostPath` volume.

```
apiVersion: v1
kind: Pod
metadata:
  name: date-tail
spec:
  containers:
  - name: date
    image: alpine
    command: ["sh", "-c"]
    args:
    - |
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
      tail -f /var/log/date-tail/output.log
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/date-tail
  volumes:
  - name: log-volume
    hostPath:
      path: /data/date-tail
  terminationGracePeriodSeconds: 0

```

This time, we changed the volume type to `hostPath` and set the path to `/data/date-tail`.

Minikube allows the use of specific paths for hostPath volumes. Valid paths include:

- /data
- /var/lib/minikube
- /var/lib/docker
- /tmp/hostpath_pv
- /tmp/hostpath-provisioner

### Deploy the Pod

```
kubectl apply -f hostpath.yaml                                                              ✔  minikube ○  11:59:32 
pod/date-tail created
```

### Check Logs

```
kubectl logs date-tail -c tail                                                            1 ✘  minikube ○  12:00:07 
Start at: Tue Jul 22 07:59:46 UTC 2025
Tue Jul 22 07:59:47 UTC 2025
Tue Jul 22 07:59:48 UTC 2025
Tue Jul 22 07:59:49 UTC 2025
Tue Jul 22 07:59:50 UTC 2025
...
```

So far, everything is working the same as with `emptyDir`.


### Recreate the Pod and Check for Persistence

Let’s forcibly recreate the Pod:

```
kubectl replace --force -f hostpath.yaml                                                    ✔  minikube ○  12:05:17 
pod "date-tail" deleted
pod/date-tail replaced
```

Now, check the logs again:

```
kubectl logs date-tail -c tail                                                              ✔  minikube ○  12:05:55 
Tue Jul 22 08:05:54 UTC 2025
Tue Jul 22 08:05:55 UTC 2025
Tue Jul 22 08:05:56 UTC 2025
Tue Jul 22 08:05:57 UTC 2025
Start at: Tue Jul 22 08:05:59 UTC 2025
Tue Jul 22 08:06:00 UTC 2025
Tue Jul 22 08:06:01 UTC 2025
Tue Jul 22 08:06:02 UTC 2025
```

This time, we can see log lines before the new `Start at` message, confirming that the data has persisted even after the Pod was deleted and recreated.

### Inspect the Files on the Host

Since `hostPath` stores data directly on the Minikube VM's filesystem, we can SSH into the VM and check:


```
minikube ssh                                                                                    ✔  12:06:15 
docker@minikube:~$ 
```

### Inside the VM:

```
minikube ssh                                                                                    ✔  12:06:15 
docker@minikube:~$ ls /data/
date-tail

docker@minikube:~$ ls /data/date-tail/
output.log

docker@minikube:~$ tail -n5 /data/date-tail/output.log
Tue Jul 22 08:08:19 UTC 2025
Tue Jul 22 08:08:20 UTC 2025
Tue Jul 22 08:08:21 UTC 2025
Tue Jul 22 08:08:22 UTC 2025
Tue Jul 22 08:08:23 UTC 2025
docker@minikube:~$ 
```

You can see that the log data is stored on the host, outside of the container lifecycle.