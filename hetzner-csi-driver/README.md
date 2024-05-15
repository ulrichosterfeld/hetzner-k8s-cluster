## Install the Kubernetes Hetzner Cloud csi-driver

Create a secret containing the Hetzner API token
```shell
nano secret.yml
```
```
# secret.yml (replace YOUR_HETZNER_API_TOKEN with your own hcloud token)
apiVersion: v1
kind: Secret
metadata:
  name: hcloud
  namespace: kube-system
stringData:
  token: YOUR_HETZNER_API_TOKEN
```
Apply the secret
```shell
kubectl apply -f secret.yml
```
Deploy the CSI driver and wait until everything is up and running.

Have a look at the Version Matrix to pick the correct version of the csi driver:

| Kubernetes | CSI Driver |                                          Deployment File                                          |
| ---------- | ---------- | ------------------------------------------------------------------------------------------------- |
|    1.29	   |   2.6.0+	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.6.0/deploy/kubernetes/hcloud-csi.yml |
|    1.28	   |   2.6.0+	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.6.0/deploy/kubernetes/hcloud-csi.yml |
|    1.27	   |   2.6.0+	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.6.0/deploy/kubernetes/hcloud-csi.yml |
|    1.26	   |   2.6.0+	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.6.0/deploy/kubernetes/hcloud-csi.yml |
|    1.25	   |   2.6.0	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.6.0/deploy/kubernetes/hcloud-csi.yml |
|    1.24	   |   2.4.0	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.4.0/deploy/kubernetes/hcloud-csi.yml |
|    1.23	   |   2.2.0	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.2.0/deploy/kubernetes/hcloud-csi.yml |
|    1.22	   |   1.6.0	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v1.6.0/deploy/kubernetes/hcloud-csi.yml |
|    1.21	   |   1.6.0	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v1.6.0/deploy/kubernetes/hcloud-csi.yml |
|    1.20	   |   1.6.0	  | https://raw.githubusercontent.com/hetznercloud/csi-driver/v1.6.0/deploy/kubernetes/hcloud-csi.yml |

In our case (Kubernetes 1.29), the correct version is 2.6.0.
```shell
kubectl apply -f https://raw.githubusercontent.com/hetznercloud/csi-driver/v2.6.0/deploy/kubernetes/hcloud-csi.yml
```
Test that the csi-driver has been successfully installed and that the hcloud-volumes storage class has been created.
```shell
kubectl -n kube-system get pods
```
```
#Output
NAME                                     READY   STATUS    RESTARTS   AGE
...
hcloud-csi-controller-7dd59b5d75-bwh5h   5/5     Running   0          14h
hcloud-csi-node-dcsn8                    3/3     Running   0          14h
hcloud-csi-node-l2kfc                    3/3     Running   0          14h
...
```
```shell
kubectl get storageclass
```
```
#Output
NAME                       PROVISIONER         RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
hcloud-volumes (default)   csi.hetzner.cloud   Delete          WaitForFirstConsumer   true                   14h
```
To verify everything is working, create a persistent volume claim and a pod which uses that volume.

Create the my-csi-app
```shell
nano my-csi-app.yml
```
```
# my-csi-app.yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: hcloud-volumes
---
kind: Pod
apiVersion: v1
metadata:
  name: my-csi-app
spec:
  containers:
    - name: my-frontend
      image: busybox
      volumeMounts:
      - mountPath: "/data"
        name: my-csi-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-csi-volume
      persistentVolumeClaim:
        claimName: csi-pvc
```
Apply the app
```shell
kubectl -n default apply -f my-csi-app.yml
```
```
#Output
persistentvolumeclaim/csi-pvc created
pod/my-csi-app created
```
Check whether the app is running
```shell
kubectl -n default get pods
```
```
#Output
NAME         READY   STATUS    RESTARTS   AGE
my-csi-app   1/1     Running   0          52s
```
List the volumes
```shell
kubectl -n default get pvc
```
```
#Output
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     VOLUMEATTRIBUTESCLASS   AGE
csi-pvc   Bound    pvc-118002a9-b5c3-4880-8521-5c9c073d07b6   10Gi       RWO            hcloud-volumes   <unset>                 2m7s
```
Connect with a app shell
```shell
kubectl exec -it my-csi-app -- /bin/sh
```
Execute the command df -h to check whether the volume is mounted
```shell
df -h
```
```
#Output
Filesystem                Size      Used Available Use% Mounted on
...
/dev/disk/by-id/scsi-0HC_Volume_100707107
                          9.7G     24.0K      9.7G   0% /data
...
/ #
```
Exit the app shell
```shell
exit
```
Delete the my-csi-app
```shell
kubectl delete pod my-csi-app
#pod "my-csi-app" deleted
```
```shell
hcloud volume list
```
```
#Output
ID          NAME                                       SIZE    SERVER   LOCATION   AGE
100707107   pvc-118002a9-b5c3-4880-8521-5c9c073d07b6   10 GB   -        nbg1       1d
```
```shell
hcloud volume delete 100707107
#Volume 100707107 deleted
```


