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
#### Nginx Deployment with NFS pvc
```

```
