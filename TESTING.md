# End to End Test of HAMi on an OpenShell Gateway

This file records the full test on a real GPU machine. Read top to bottom. Every command and every output is real, copied straight from the terminal session of 2026 April 29.

## 1. The host

The machine is one Nebius node called `nebius-tarantula`. One NVIDIA L40S with 46068 MB of memory. Driver 580.126.09. Ubuntu host with kernel 6.11.0.

```shell
$ nvidia-smi
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
|   0  NVIDIA L40S                    On  |   00000000:8D:00.0 Off |                    0 |
| N/A   25C    P8   34W /  325W           |       0MiB /  46068MiB |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```

```shell
$ which nvidia-container-runtime
/usr/bin/nvidia-container-runtime

$ nvidia-container-runtime --version
NVIDIA Container Runtime version 1.18.2
```

The host already runs a K3s cluster (v1.34) outside of OpenShell. That cluster was used earlier in the day to confirm HAMi alone works as the upstream documentation says. Those numbers (10 vGPU slices, two pods sharing one card with 8 GB each) are not what this test is about. This test is about what happens when a sandbox is created through the OpenShell platform path.

## 2. Install OpenShell

The OpenShell CLI is downloaded from the published install script.

```shell
$ curl -fsSL https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh -o /tmp/openshell-install.sh
$ sh /tmp/openshell-install.sh
openshell: resolving latest version...
openshell: downloading openshell v0.0.36 (x86_64-unknown-linux-musl)...
openshell: verifying checksum...
openshell: extracting...
openshell: installed openshell 0.0.36 to /home/ubuntu/.local/bin/openshell

$ openshell --version
openshell 0.0.36
```

## 3. Start the gateway with GPU support

The user account first needs to be in the `docker` group.

```shell
$ sudo usermod -aG docker ubuntu
```

Then start the gateway. The `--gpu` flag tells OpenShell to enable the NVIDIA GPU pathway inside the gateway K3s cluster. The flag has no option to choose a different GPU driver. OpenShell only knows about the upstream NVIDIA k8s device plugin today.

```shell
$ openshell gateway start --gpu
Deploying local gateway openshell...
  Checking Docker
  Downloading gateway
  Initializing environment
```

After about three minutes the gateway container is healthy.

```shell
$ docker ps --format "{{.Names}} {{.Status}}"
openshell-cluster-openshell  Up About a minute (healthy)
```

The K3s cluster inside the container has the OpenShell server, the agent sandbox controller, and the nvidia device plugin already installed by the bootstrap.

```shell
$ kubectl get pods -A
NAMESPACE              NAME                                      READY   STATUS      RESTARTS   AGE
agent-sandbox-system   agent-sandbox-controller-0                1/1     Running     0          2m57s
kube-system            coredns-c4dbffb5f-vb5j4                   1/1     Running     0          2m57s
kube-system            helm-install-nvidia-device-plugin-n9krl   0/1     Completed   0          2m56s
kube-system            local-path-provisioner-5c4dc5d66d-7kvqz   1/1     Running     0          2m57s
kube-system            metrics-server-786d997795-kgcbc           1/1     Running     0          2m57s
nvidia-device-plugin   nvidia-device-plugin-vccqr                1/1     Running     0          5s
openshell              openshell-0                               1/1     Running     0          2m35s
```

(The nvidia device plugin pod first crashed with `failed to create FS watcher: too many open files`. Fixed with `sudo sysctl -w fs.inotify.max_user_instances=1024 fs.inotify.max_user_watches=1048576`. Worth noting for the upstream report. This is a host setting, not an OpenShell bug.)

The node now reports one whole GPU as one allocatable unit, which is the upstream behaviour.

```shell
$ kubectl get node openshell-openshell -o json | jq '.status.allocatable["nvidia.com/gpu"]'
"1"
```

## 4. Baseline. OpenShell sandbox with no HAMi

```shell
$ openshell sandbox create --gpu --name baseline-gpu
```

After about two minutes the sandbox is `Running`.

```shell
$ kubectl get pod -n openshell baseline-gpu -o json | jq '{runtime: .spec.runtimeClassName, limits: .spec.containers[0].resources.limits}'
{
  "runtime": null,
  "limits": {
    "nvidia.com/gpu": "1"
  }
}
```

The pod has no `runtimeClassName` and asks for one GPU. Inside the sandbox the application sees the full physical card.

```shell
$ kubectl exec -n openshell baseline-gpu -c agent -- nvidia-smi
+-----------------------------------------+------------------------+
|   0  NVIDIA L40S                    On  |   00000000:8D:00.0 Off |
| N/A   26C    P8   34W /  325W           |       0MiB /  46068MiB |
+-----------------------------------------+------------------------+
```

Full 46068 MiB. That is the expected baseline.

## 5. Install HAMi into the gateway cluster

HAMi takes over the `nvidia.com/gpu` resource and exposes it as N slices. The chart needs three values for this cluster.

```shell
$ kubectl label node openshell-openshell gpu=on --overwrite
node/openshell-openshell labeled

$ helm repo add hami-charts https://project-hami.github.io/HAMi/
$ helm repo update
$ helm install hami hami-charts/hami \
    --namespace kube-system \
    --set devicePlugin.deviceSplitCount=10 \
    --set scheduler.kubeScheduler.imageTag=v1.35.3 \
    --set devicePlugin.runtimeClassName=nvidia
NAME: hami
LAST DEPLOYED: Wed Apr 29 18:46:14 2026
STATUS: deployed
```

The three values matter.

`deviceSplitCount=10` asks HAMi to expose each physical GPU as 10 logical slices.

`scheduler.kubeScheduler.imageTag=v1.35.3` matches the kube version inside the OpenShell K3s. HAMi ships its own kube scheduler binary and the minor must match.

`devicePlugin.runtimeClassName=nvidia` makes HAMi run its own device plugin under the nvidia container runtime. Without this the plugin fails on `Incompatible strategy detected auto`.

A few minutes later both HAMi pods are up and the node now reports 10 allocatable slices.

```shell
$ kubectl get pods -n kube-system -l app.kubernetes.io/instance=hami
NAME                              READY   STATUS    RESTARTS   AGE
hami-device-plugin-8fw69          2/2     Running   0          2m
hami-scheduler-7bbbfc4f7f-d6n29   2/2     Running   0          1m

$ kubectl get node openshell-openshell -o json | jq '.status.allocatable["nvidia.com/gpu"]'
"10"
```

Note. On a single node cluster the HAMi scheduler Deployment ships with `RollingUpdate` and a hard pod anti affinity. Any helm upgrade deadlocks. Patch to `Recreate` once after install.

```shell
$ kubectl patch deployment hami-scheduler -n kube-system --type=json \
    -p='[{"op":"replace","path":"/spec/strategy","value":{"type":"Recreate"}}]'
deployment.apps/hami-scheduler patched
```

## 6. Create an OpenShell GPU sandbox under HAMi

This is the real test. Delete the baseline sandbox and create a new one.

```shell
$ openshell sandbox delete baseline-gpu
✓ Deleted sandbox baseline-gpu

$ openshell sandbox create --gpu --name hami-test
```

Once the new sandbox is `Running` look at the pod spec.

```shell
$ kubectl get pod -n openshell hami-test -o json | jq '{runtime: .spec.runtimeClassName, limits: .spec.containers[0].resources.limits, hami_annotations: .metadata.annotations}'
{
  "runtime": "nvidia",
  "limits": {
    "nvidia.com/gpu": "1",
    "nvidia.com/gpucores": "100"
  },
  "hami_annotations": {
    "hami.io/bind-phase": "success",
    "hami.io/bind-time": "1777488587",
    "hami.io/vgpu-devices-allocated": "GPU-7b976500-49f8-f067-8dab-99e9ddc64b30,NVIDIA,46068,100:;",
    "hami.io/vgpu-node": "openshell-openshell"
  }
}
```

Two surprises in this output.

First, the pod has `runtimeClassName: nvidia`. OpenShell did not set this. The HAMi mutating admission webhook injected it after the pod was submitted. So the runtime class question is already solved by HAMi without any change in OpenShell.

Second, the `vgpu-devices-allocated` annotation reads `..., 46068, 100`. HAMi gave this single sandbox the full 46068 MB of GPU memory and 100 percent of the cores. The reason is in the resource limits. The pod asked for `nvidia.com/gpu: 1` and nothing else. HAMi treats a missing memory request as "give the whole card".

The `nvidia-smi` output inside the sandbox confirms it.

```shell
$ kubectl exec -n openshell hami-test -c agent -- nvidia-smi
+-----------------------------------------+------------------------+
|   0  NVIDIA L40S                    On  |   00000000:8D:00.0 Off |
| N/A   26C    P8   34W /  325W           |       0MiB /  46068MiB |
+-----------------------------------------+------------------------+
```

Still 46068 MiB. Same as the baseline. HAMi is technically active but it is enforcing nothing, because nothing was asked of it.

## 7. Try a second pod that does ask for memory

Now apply a manual pod that asks for 8000 MB of GPU memory.

```shell
$ kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: hami-with-mem
spec:
  runtimeClassName: nvidia
  restartPolicy: Never
  containers:
  - name: cuda
    image: nvcr.io/nvidia/cuda:12.4.1-base-ubuntu22.04
    command: ["bash", "-c", "nvidia-smi && sleep 60"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: "8000"
EOF
pod/hami-with-mem created
```

The pod stays `Pending`. The HAMi scheduler explains why.

```shell
$ kubectl describe pod hami-with-mem
Events:
  Warning  FilteringFailed  hami-scheduler  1 nodes CardInsufficientMemory(openshell-openshell)
  Warning  FilteringFailed  hami-scheduler  no available node, 1 nodes do not meet
```

`CardInsufficientMemory`. The OpenShell sandbox is holding all 46068 MB. There is no room left on the only GPU on the node. Sharing is dead.

## 8. Confirm HAMi works when memory is requested

Delete the OpenShell sandbox so the device frees up.

```shell
$ openshell sandbox delete hami-test
✓ Deleted sandbox hami-test
```

The `hami-with-mem` pod schedules within seconds and runs.

```shell
$ kubectl get pod hami-with-mem
NAME            READY   STATUS    RESTARTS   AGE
hami-with-mem   1/1     Running   0          5s

$ kubectl logs hami-with-mem
+-----------------------------------------+------------------------+
|   0  NVIDIA L40S                    On  |   00000000:8D:00.0 Off |
| N/A   26C    P8   34W /  325W           |       0MiB /   8000MiB |
+-----------------------------------------+------------------------+
```

Reports 8000 MiB. Same physical card (Bus-ID 00000000:8D:00.0). HAMi is enforcing the slice exactly as documented.

## 9. What this test proves

The bridge from OpenShell to HAMi is one missing field on the sandbox specification.

If a sandbox could be created with a memory request, for example `openshell sandbox create --gpu --gpu-memory 8gb`, then the OpenShell driver would render `nvidia.com/gpumem: 8000` into the pod spec and HAMi would do its job. Multiple sandboxes would coexist on the same physical card with hard memory limits enforced by the runtime.

Without the field there is no way to share the card. The first sandbox eats all of it. Every later sandbox is blocked at the scheduler.

Other facts that the test cleared up.

The pod runtime class is not a problem. HAMi sets it through the mutating webhook even when OpenShell leaves it null. No code change is needed in `openshell-driver-kubernetes` for runtime class.

The K3s containerd config inside the OpenShell gateway image already has `nvidia` registered as a runtime. The host nvidia container toolkit is detected and used.

The CDI strategy that the upstream OpenShell nvidia chart uses (`deviceListStrategy=cdi-cri`) does not conflict with HAMi as long as only one of the two device plugins is active per cluster. In this test HAMi was installed second and overrode the resource registration. A clean install of HAMi from the start would be cleaner.

The actual proposal for OpenShell is in PROPOSAL.md.
