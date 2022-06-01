## KAHU Metadata Backup POC

This is a demo of kahu metadata backup using NFS Provider.

## POC Introduction
KAHU is a project which focuses on backup and restore features for containerized applications.

Project link : https://github.com/soda-cdm/kahu

At highlevel, Kahu aims to handle backup and restore of metadata and persistent volumes used by the applications.
This POC focuses of realizing the metadata backup of an application using NFS provider.

The E2E flow looks as shown below
  ![](/kahu/MetadataBackupPoc/E2E_Flow.png)
  
  ## Environment Details
  OS version: ubuntu 18.0.4
  
  K8S Version: 1.24
  
  go version: go1.17.2
  

## Execution steps

Note: This POC uses Kind for kubernetes cluster setup. 

Refer https://kind.sigs.k8s.io/docs/user/quick-start/ for details of setup

However kubernetes cluster can be setup in any other way.

POC execution steps:

* Download code from poc branch
```
$ git clone https://github.com/soda-cdm/kahu.git
$ cd kahu
$ git checkout metadata_backup_poc 
```

* Build binaries of all the necessary components
```
$ CGO_ENABLED=0 go build -o _output/bin/kahuserver cmd/kahu/kahuserver.go
$ CGO_ENABLED=0 go build -o _output/bin/metaservice cmd/metaservice/main.go 
$ CGO_ENABLED=0 go build -o _output/bin/nfsprovider providers/nfs_provider/cmd/main.go
```

* Build docker image which contains all the above binaries
```
$ docker build -t kahu:v1 .
```

* Upload image to all nodes in kind cluster. This step may vary based on environment used
```
$ kind load docker-image kahu:v1
```

* Setup NFS server. This is an external entity running which can be connected from kahu environment.
```
$ apt update
$ apt install nfs-kernel-server
$ mkdir -p /mnt/nfs_share
$ chown -R nobody:nogroup /mnt/nfs_share/
$ chmod 777 /mnt/nfs_share/
$ vim /etc/exports
$ exportfs -a
$ systemctl restart nfs-kernel-server
$ ufw status
$ ufw enable
$ ufw status
$ ufw disable
$ ufw status
```
* Deploy kahu specific CRDs
```
$ kubectl apply -f config/
```

* Deploy NFS Provider. Ensure to update nfs server and path parameters in below yaml file based on the NFS server setup
```
$ kubectl apply -f providers/nfs_provider/deploy/nfs-provider-deployment.yaml
```

* Deploy kahu backup controller 
```
$ kubectl apply -f deploy/backup-controller.yaml
```

* Create a sample application yaml (sample_pod.yaml) and deploy. This POD will be considered for metadata backup
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```
```
kubectl apply -f sample_pod.yaml
```

* Now create kahu backup object specifying that pod resouirce to be backed up. Verify the creation of backup object
```
$ kubectl apply -f examples/backup.yaml
$ kubectl get backups
```

* Finally verify that pod manifests backup is taken at the NFS server side. Execute below commands at NFS server
```
$ cd /mnt/nfs_share/
$ ls
```



