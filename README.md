# NFS-Static-and-Dynamic-PVC

## Install NFS Server
```
sudo systemctl status nfs-server
sudo apt install nfs-kernel-server nfs-common portmap
sudo start nfs-server
mkdir -p /share 
chmod -R 777 /share # for simple use but not advised
```

## Validate mount point
```
sudo vi /etc/exports
/share  *(rw,sync,no_subtree_check,no_root_squash,insecure)
$ sudo exportfs -rv
exporting *:/share
$ showmount -e
/share  *
```

## Mount on worker node
```
mount -t nfs 10.10.0.243:/share /mnt
$ mount | grep share
10.10.0.243:/share on /mnt type nfs4
$ exit
```

## Static NFS PVC

#### Create NFS PVC storage Class
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    name: mynfs # name can be anything
spec:
  storageClassName: manual # same storage class as pvc
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.10.0.243 # ip addres of nfs server
    path: "/share" # path to directory
```

#### Create NFS PVC Claim
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    name: mynfs # name can be anything
spec:
  storageClassName: manual # same storage class as pvc
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.10.0.243 # ip addres of nfs server
    path: "/share" # path to directory
platypus@platypus01:~/nfspvc$ ls
nfs-claim.yaml  nfs.yaml  nginx-deployment.yaml
platypus@platypus01:~/nfspvc$ cat nfs-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany #  must be the same as PersistentVolume
  resources:
    requests:
      storage: 1Gi
```

#### Nginx Deployment with NFS pvc
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nfs-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
      - name: nfs-test
        persistentVolumeClaim:
          claimName: nfs-pvc # same name of pvc that was created
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nfs-test # name of volume should match claimName volume
          mountPath: /usr/share/nginx/html # mount inside of contianer
```

## Dynamic Provisioner NFS PVC
#### Creating RBAC Service Account
```
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-pod-provisioner-sa
---
kind: ClusterRole # Role of kubernetes
apiVersion: rbac.authorization.k8s.io/v1 # auth API
metadata:
  name: nfs-provisioner-clusterRole
rules:
  - apiGroups: [""] # rules on persistentvolumes
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-provisioner-rolebinding
subjects:
  - kind: ServiceAccount
    name: nfs-pod-provisioner-sa # defined on top of file
    namespace: default
roleRef: # binding cluster role to service account
  kind: ClusterRole
  name: nfs-provisioner-clusterRole # name defined in clusterRole
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-pod-provisioner-otherRoles
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-pod-provisioner-otherRoles
subjects:
  - kind: ServiceAccount
    name: nfs-pod-provisioner-sa # same as top of the file
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: nfs-pod-provisioner-otherRoles
  apiGroup: rbac.authorization.k8s.io
```

#### Create NFS storage Class
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storageclass # IMPORTANT pvc needs to mention this name
provisioner: nfs-test # name can be anything
parameters:
  archiveOnDelete: "false"
```

#### Create NFS client provisioner
```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-pod-provisioner
spec:
  selector:
    matchLabels:
      app: nfs-pod-provisioner
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-pod-provisioner
    spec:
      serviceAccountName: nfs-pod-provisioner-sa # name of service account created in rbac.yaml
      containers:
        - name: nfs-pod-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-provisioner-v
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME # do not change
              value: nfs-test # SAME AS PROVISONER NAME VALUE IN STORAGECLASS
            - name: NFS_SERVER # do not change
              value: 10.10.0.243 # Ip of the NFS SERVER
            - name: NFS_PATH # do not change
              value: /share # path to nfs directory setup
      volumes:
       - name: nfs-provisioner-v # same as volumemouts name
         nfs:
           server: 10.10.0.243
           path: /share
```

#### Requesting a PV claim
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-test
spec:
  storageClassName: nfs-storageclass # SAME NAME AS THE STORAGECLASS
  accessModes:
    - ReadWriteMany #  must be the same as PersistentVolume
  resources:
    requests:
      storage: 500Mi
```

#### Create a Nginx Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nfs-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
      - name: nfs-test #
        persistentVolumeClaim:
          claimName: nfs-pvc-test  # same name of pvc that was created
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nfs-test # name of volume should match claimName volume
          mountPath: share # mount inside of contianer
```

## Reference article
```
https://medium.com/@myte/kubernetes-nfs-and-dynamic-nfs-provisioning-97e2afb8b4a9
```
