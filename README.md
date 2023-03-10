# Install KubeVirt and Deploy VMs on Kubernetes

## Requirements
- Kubernetes Cluster
- Kubernetes apiserver must have --allow-privileged=true
- kubectl

## Content
- Validate Hardware Support
- Install KubeVirt
- Install Virctl
- Install CDI
- Create PVC
- Create Virtual Machine
- Expose VMI become Services

### Validate Hardware Support
```
$ sudo virt-host-validate qemu
  QEMU: Checking for hardware virtualization                                 : PASS
  QEMU: Checking if device /dev/kvm exists                                   : PASS
  QEMU: Checking if device /dev/kvm is accessible                            : PASS
  QEMU: Checking if device /dev/vhost-net exists                             : PASS
  QEMU: Checking if device /dev/net/tun exists                               : PASS
```

### Installing KubeVirt
```
export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml
kubectl get pods -n kubevirt
kubectl -n kubevirt wait kv kubevirt --for condition=Available
```

### Install Virctl
```
VERSION=$(kubectl get kubevirt.kubevirt.io/kubevirt -n kubevirt -o=jsonpath="{.status.observedKubeVirtVersion}")
ARCH=$(uname -s | tr A-Z a-z)-$(uname -m | sed 's/x86_64/amd64/') || windows-amd64.exe
echo ${ARCH}
curl -L -o virtctl https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-${ARCH}
chmod +x virtctl
sudo install virtctl /usr/local/bin
```

### Install Containerized Data Importer (CDI)
```
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/v1.48.1/cdi-operator.yaml
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/v1.48.1/cdi-cr.yaml
kubectl get cdi cdi -n cdi
kubectl get pods -n cdi
```

### Create PVC
Fill storageClassName with your NFS StorageClass
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "fedora"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "https://download.fedoraproject.org/pub/fedora/linux/releases/36/Cloud/x86_64/images/Fedora-Cloud-Base-36-1.5.x86_64.raw.xz"
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: your-nfs-storage-class
```

### Check PVC, Pods and Log
```
kubectl get pvc fedora -o yaml
kubectl get pod
kubectl logs -f importer-fedora-pnbqh
```

### Create Virtual Machine
```
vim vm1_pvc.yml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  creationTimestamp: 2018-07-04T15:03:08Z
  generation: 1
  labels:
    kubevirt.io/os: linux
  name: fedora-vm
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        kubevirt.io/domain: vm1
    spec:
      domain:
        cpu:
          cores: 2
        devices:
          disks:
          - disk:
              bus: virtio
            name: disk0
          - cdrom:
              bus: sata
              readonly: true
            name: cloudinitdisk
          interfaces:
          - name: default
            masquerade: {}
        resources:
          requests:
            memory: 1024M
      networks:
      - name: default
        pod: {}
      volumes:
      - name: disk0
        persistentVolumeClaim:
          claimName: fedora
      - cloudInitNoCloud:
          userData: |
            #cloud-config
            hostname: vm1
            ssh_pwauth: True
            disable_root: false
            ssh_authorized_keys:
            - ssh-rsa YOUR_SSH_PUB_KEY_HERE
        name: cloudinitdisk
```
Inject ssh key with ssh pubkey
```
ssh-keygen
PUBKEY=`cat ~/.ssh/id_rsa.pub`
sed -i "s%ssh-rsa.*%$PUBKEY%" vm1_pvc.yml
kubectl create -f vm1_pvc.yml
```
```
kubectl get vmi
NAME         AGE   PHASE     IP           NODENAME   READY
fedora-vm    75m   Running   10.16.0.91   worker02   True
```

### Expose VMI become Services
```
kubectl expose vmi fedora-vm --name=fedora-svc --port=20222 --target-port=22 --type=NodePort
kubectl get svc
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
fedora-svc        NodePort    10.103.24.183   10.8.60.62    4242:2022      1m
```
Ssh into vm
```
ssh -i ~/.ssh/id_rsa fedora@your-nodeport-ip -p 4242
Last login: Sat Mar  4 11:22:59 2023 from 10.7.91.5
[fedora@fedora-vm ~]$
```

## Ref
- https://kubevirt.io/user-guide/operations/installation/
- https://medium.com/adessoturkey/create-a-windows-vm-in-kubernetes-using-kubevirt-b5f54fb10ffd
- https://www.kubermatic.com/blog/bringing-your-vms-to-kubernetes-with-kubevirt/