# 6NRP Kubernetes Tutorial

**Scheduling in Kubernetes\
Hands on session**

## Explore the system

Let's start with looking what's available in the the system.

Start by listing all the available nodes:

```
kubectl get nodes
```

Let's now dig a little deeper. For example, you can see which nodes have which GPU type:

```
kubectl get nodes -L nvidia.com/gpu.product
```

Now, pick one node, and see what other resources it has:

```
kubectl get nodes -o yaml <nodename>
```

See how many CPU cores and how much memory it has.

If you picked a node with a GPU, look for the "nvidia.com/gpu.product" in the output. Did it match?

Do you see any other interesting label you may want to match on?

Now pick another attribute from the output, and use it instead of gpu.product in the above all-node query. Did it return what you expected?

## Validating requirements

In the next section we will play with adding requirements to Pod yamls. But first let's make sure those resources are actually available.

As a simple example, let's pick a specific GPU type:

```
kubectl get node -l 'nvidia.com/gpu.product=NVIDIA-GeForce-GTX-1080-Ti'
```

Did you get any hits?

Here we look for nodes with 100Gbps NICs:

```
kubectl get node -l 'nautilus.io/network=100000'
```

Did you get any hits?

How about a negative selector? And let's see what do we get:

```
kubectl get node -l 'nvidia.com/gpu.product!=NVIDIA-GeForce-GTX-1070,nvidia.com/gpu.product!=NVIDIA-GeForce-GTX-1080' -L nvidia.com/gpu.product | sort -k 6
```

## Requirements in pods

You have already used requirements in the pods. Here is the very first pod we made you launch:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-<username>
spec:
  containers:
  - name: mypod
    image: busybox:latest
    resources:
      limits:
        memory: 1Gi
        cpu: 1
      requests:
        memory: 100Mi
        cpu: 100m
    command: ["sleep", "10000"]
```

But we set them to be really low, so it was virtually guaranteed that the Pod would start.

Let's make it a job and raise the requirements drastically:


```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: jb1-<username>
spec:
  completions: 1
  ttlSecondsAfterFinished: 1800
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: mypod
        image: rockylinux:8
        resources:
          limits:
            memory: 800Gi
            cpu: 110
          requests:
            memory: 800Gi
            cpu: 110
        command: ["sh", "-c", "df -H /dev/shm; sleep 10"]
        volumeMounts:
        - name: ramdisk1
          mountPath: /dev/shm
      volumes:
      - name: ramdisk1
        emptyDir:
          medium: Memory
```

Create the pod, and check its status.
```
kubectl get pods -o wide
```

Since we were so greedy, this pod will never start.

Let's relax the requirement, but keep the limits high:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: jb2-<username>
spec:
  completions: 1
  ttlSecondsAfterFinished: 1800
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: mypod
        image: rockylinux:8
        resources:
          limits:
            memory: 12Gi
            cpu: 12
          requests:
            memory: 10Gi
            cpu: 10
        command: ["sh", "-c", "df -H /dev/shm; sleep 10"]
        volumeMounts:
        - name: ramdisk1
          mountPath: /dev/shm
      volumes:
      - name: ramdisk1
        emptyDir:
          medium: Memory
```

The pod should now start.

Since we are mounting the ramdisk, we can now check how much memory we actually got:
```
kubectl logs jb2-<username>-<hash>
```

Was it what you expected?

After you are done exploring, destroy the job(s).

Let's now ask for a different kind of resources; let's ask for a GPU. We also change the container, so that we get the proper NVIDIA tools in place.

**Note:** While you can ask for a fraction of a CPU, you cannot ask for a fraction of a GPU in our current setup. You should also keep the same number for requirements and limits.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gp1-<username>
spec:
  containers:
  - name: mypod
    image: nvidia/cuda:11.7.1-runtime-ubuntu20.04
    resources:
      limits:
        memory: 1Gi
        cpu: 1
        nvidia.com/gpu: 1
      requests:
        memory: 500Mi
        cpu: 100m
        nvidia.com/gpu: 1
    command: ["sh", "-c", "sleep 1000"]
```

Once the pod started, login using kubectl exec and check what kind of GPU you got:

```
nvidia-smi -L
```

You can see which node you landed on with
```
kubectl get pods -o wide
```

Then query the attributes of that node. Do they match?

After you are done exploring, destroy the pod(s).

## Complex requirements in pods

Now, let's add a more complex requirement.
Let's restrict what GPU model are we willing to use.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gp2-<username>
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nvidia.com/gpu.product
            operator: In
            values:
            - NVIDIA-GeForce-GTX-1070
            - NVIDIA-GeForce-GTX-1080-Ti
  containers:
  - name: mypod
    image: nvidia/cuda:11.7.1-runtime-ubuntu20.04
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
        nvidia.com/gpu: 1
      requests:
        memory: 500Mi
        cpu: 100m
        nvidia.com/gpu: 1
    command: ["sh", "-c", "sleep 1000"]
```

Did the pod start?

Check what node it is on and what GPU it is supposed to have.

Log into the Pod and check that you indeed got the desired GPU type.

After you are done exploring, destroy the pod(s).

## Preferences in pods

Sometimes you would prefer something, but can live with less. In this example, let's ask for the fastest GPU in out pool, but not as a hard requirement:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gp3-<username>
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: nvidia.com/gpu.product
            operator: In
            values:
            - NVIDIA-A100-PCIE-40GB-MIG-2g.10gb
            - Tesla-T4
  containers:
  - name: mypod
    image: nvidia/cuda:11.7.1-runtime-ubuntu20.04
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
        nvidia.com/gpu: 1
      requests:
        memory: 500Mi
        cpu: 100m
        nvidia.com/gpu: 1
    command: ["sh", "-c", "sleep 1000"]
```

Now check you Pod. How long did it take for it to start? What GPU type did you get.

*Note:* You can combine requirements with preferences. Please do try, if you have any time left.

After you are done exploring, remember to destroy the pod.

## The end

We do not include hands-on exercises based on priorities, as they are very non-deterministic.
Hopefully the sides provided clear enough instructions for you to try it at home on your own cluster.

Please make sure you did not leave any jobs or pods behind.
