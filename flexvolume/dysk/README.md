# dysk flex volume driver for Kubernetes (Preview)
 - supported Kubernetes version: v1.8, v1.9
 - supported agent OS: Linux 

# About
[dysk - Fast kernel-mode mount/unmount of AzureDisk](https://github.com/khenidak/dysk)

# Prerequisite
User needs to create an storage account in the same region as the kubernetes cluster and provide storage account name and account key in below example.

# Install dysk driver on a kubernetes cluster
## 1. config kubelet service (skip this step in [AKS](https://azure.microsoft.com/en-us/services/container-service/) or from [acs-engine](https://github.com/Azure/acs-engine) v0.12.0)
specify `volume-plugin-dir` in kubelet service config 
```
sudo vi /etc/systemd/system/kubelet.service
  --volume=/etc/kubernetes/volumeplugins:/etc/kubernetes/volumeplugins:rw \
        --volume-plugin-dir=/etc/kubernetes/volumeplugins \
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Note:
 - `/etc/kubernetes/volumeplugins` has already been the default flexvolume plugin directory in acs-engine (starting from v0.12.0)
 - There would be one line of [kubelet log](https://github.com/andyzhangx/Demo/tree/master/debug#q-how-to-get-k8s-kubelet-logs-on-linux-agent) in agent node like below showing that `flexvolume-azure/dysk` is loaded correctly
```
I0122 08:24:47.761479    2963 plugins.go:469] Loaded volume plugin "flexvolume-azure/dysk"
```
 - Flexvolume is GA from Kubernetes **1.8** release, v1.7 is depreciated since it does not support [Dynamic Plugin Discovery](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md#dynamic-plugin-discovery).
 
## 2. install dysk flex volume driver on every agent node
### Option#1. Automatically install by k8s daemonset
create daemonset to install dysk driver
 - v1.9
```
kubectl create -f https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/dysk/deployment/dysk-flexvol-installer-1.9.yaml
```
 - v1.8
```
 kubectl create -f https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/dysk/deployment/dysk-flexvol-installer-1.8.yaml
```
 - check daemonset status:
```
kubectl describe daemonset dysk-flexvol-installer --namespace=flex
kubectl get po --namespace=flex
```

### Option#2. Manually install on every agent node (depreciated)
Take k8s v1.9 as an example:
```
version=v1.9
sudo mkdir -p /etc/kubernetes/volumeplugins/azure~dysk/bin
cd /etc/kubernetes/volumeplugins/azure~dysk/bin

sudo wget -O dysk https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/dysk/binary/hyperkube-$version/v0.2.4/dysk
sudo chmod a+x dysk

cd /etc/kubernetes/volumeplugins/azure~dysk
sudo wget -O dysk https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/dysk/dysk
sudo chmod a+x dysk
```

# Basic Usage
## 1. create a secret which stores dysk account name and password
```
kubectl create secret generic dyskcreds --from-literal username=USERNAME --from-literal password="PASSWORD" --type="azure/dysk"
```

## 2. create a pod with dysk flexvolume mount on linux
#### Option#1 Use flexvolume mount directly inside a pod
- download `nginx-flex-dysk.yaml` file and modify `container` field
```
wget -O nginx-flex-dysk.yaml https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/dysk/nginx-flex-dysk.yaml
vi nginx-flex-dysk.yaml
```
 - create a pod with dysk flexvolume driver mount
```
kubectl create -f nginx-flex-dysk.yaml
```

#### Option#2 Create dysk flexvolume PV & PVC and then create a pod based on PVC
 - download `pv-dysk-flexvol.yaml` file, modify `container` field and create a dysk flexvolume persistent volume(PV)
```
wget https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/dysk/pv-dysk-flexvol.yaml
vi pv-dysk-flexvol.yaml
kubectl create -f pv-dysk-flexvol.yaml
```

 - create a dysk flexvolume persistent volume claim(PVC)
```
 kubectl create -f https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/dysk/pvc-dysk-flexvol.yaml
```

 - check status of PV & PVC until its Status changed from `Pending` to `Bound`
 ```
 kubectl get pv
 kubectl get pvc
 ```
 
 - create a pod with dysk flexvolume PVC
```
 kubectl create -f https://raw.githubusercontent.com/andyzhangx/kubernetes-drivers/master/flexvolume/dysk/nginx-flex-dysk-pvc.yaml
 ```

 - watch the status of pod until its Status changed from `Pending` to `Running`
```
watch kubectl describe po nginx-flex-dysk
```

## 3. enter the pod container to do validation
kubectl exec -it nginx-flex-dysk -- bash

```
root@nginx-flex-dysk:/# df -h
Filesystem         Size  Used Avail Use% Mounted on
overlay            291G  6.3G  285G   3% /
tmpfs              3.4G     0  3.4G   0% /dev
tmpfs              3.4G     0  3.4G   0% /sys/fs/cgroup
/dev/dyskpoosf9g0  992M  1.3M  924M   1% /data
/dev/sda1          291G  6.3G  285G   3% /etc/hosts
shm                 64M     0   64M   0% /dev/shm
tmpfs              3.4G   12K  3.4G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs              3.4G     0  3.4G   0% /sys/firmware
```
In the above example, there is a `/data` directory mounted as dysk filesystem.

### Tips
##### How to use flexvolume driver in Helm
Since flexvolume does not support dynamic provisioning, storageClass should be set as empty in Helm chart, take [wordpress](https://github.com/kubernetes/charts/tree/master/stable/wordpress) as an example:
 - Set up a dysk flexvolume PV and also `dyskcreds` first
```
kubectl create secret generic dyskcreds --from-literal username=USERNAME --from-literal password="PASSWORD" --type="azure/dysk"
kubectl create -f pv-dysk-flexvol.yaml
```
 - Specify `persistence.accessMode=ReadWriteMany,persistence.storageClass="-"` in [wordpress](https://github.com/kubernetes/charts/tree/master/stable/wordpress) chart
```
helm install --set persistence.accessMode=ReadWriteMany,persistence.storageClass="-" stable/wordpress
```

### Links
[Flexvolume doc](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md)

More clear steps about flexvolume by Redhat doc: [Persistent Storage Using FlexVolume Plug-ins](https://docs.openshift.org/latest/install_config/persistent_storage/persistent_storage_flex_volume.html)

[dysk - Fast kernel-mode mount/unmount of AzureDisk](https://github.com/khenidak/dysk)