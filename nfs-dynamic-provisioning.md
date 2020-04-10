# NFS dynamic provisioning

아직까진 공식적으로 (v1.17) 쿠버네티스에서 NFS Provisioner 를 제공해주지 않는다. 인큐베이터 프로젝트 ([https://github.com/kubernetes-incubator/external-storage) 를 사용하여 dynamic provisioning 하는 방법을 알아본다.


## StaticProvisioning 이란?
- 기존에 사용할 수 있는 볼륨(PV, Persistent Volume)을 미리 만들어놔야 요청(PVC, Persistent Volume Claim)이 들어올 때 할당이 됨  
- 파드에서는 PVC를 마운트해서 사용  
- 운영자는 PVC가 있을 때마다 PV를 만들어줘야 함  
- 미리 PV를 만들어놔도 되지만 자원을 낭비 함

##  Dynamic Provisioning 이란?

- 파드가 만들어 질 때 필요한 볼륨을 요청(PVC, Persistent Volume Claim) 하면 자동으로 생성 됨
- storageClass라는 쿠버네티스 오브젝트를 통해 볼륨을 동적으로 할당받음

> gke에서 storage 예시
``` bash
bigdata@datateam-server5:~/kubernetes/external-storage/nfs-client/deploy$ cat << EOF | k create -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pvc1
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  zone: europe-west1-b
EOF
```
- provisioner: 항목에 사용하고 싶은 volume의 provisioner를 적으면 만들어 짐  
 - NFS는 Kubernetes에서 기본으로 제공하는 provisioner가 없으므로 따로 배포해서 사용해야 함


## nfs dynamic provisioner 배포방법
1. code clone
	``` bash
	bigdata@datateam-server5:~/kubernetes$ git clone https://github.com/kubernetes-incubator/external-storage.git
	```
2. 네임스페이스 설정 및 RBAC 만들기
	``` bash
	bigdata@datateam-server5:~/kubernetes$ cd external-storage/nfs-client/deploy
	$ sed -i "s/namespace: default/namespace: <사용할 네임스페이스>/g" rbac.yaml deployment.yaml
	$ k create -f rbac.yaml
	```
3. NFS 서버 설정 및 배포하기
	```	 yaml
	kind: Deployment  
	apiVersion: apps/v1  
	metadata:  
	  name: bigdata-nfs-client-provisioner  
	spec:  
	  replicas: 1  
	  strategy:  
	    type: Recreate  
	  selector:  
	    matchLabels:  
	      app: bigdata-nfs-client-provisioner  
	  template:  
	    metadata:  
	      labels:  
	        app: bigdata-nfs-client-provisioner  
	    spec:  
	      serviceAccountName: nfs-client-provisioner  
	      containers:  
	        - name: bigdata-nfs-client-provisioner  
	          image: quay.io/external_storage/nfs-client-provisioner:latest  
	          volumeMounts:  
	            - name: nfs-client-root  
	              mountPath: /persistentvolumes  
	          env:  
	            - name: PROVISIONER_NAME  
	              value: bigdata-provisioner
	            - name: NFS_SERVER  
	              value: <NFS_SERVER Addr> ex) fs.gamggi.io
	            - name: NFS_PATH  
	              value: <NFS_SERVER path> ex) /bigdata
	      nodeSelector:  
	        kubernetes.io/hostname: datateam-server3   
	      volumes:  
	        - name: nfs-client-root  
	          nfs:  
	            server: <NFS_SERVER Addr> ex) fs.gamggi.io  
	            path: <NFS_SERVER path> ex) /bigdata
	```
	
## NFS dynamic provisioner 사용하기
1. StorageClass 만들기
	``` yaml
	apiVersion: storage.k8s.io/v1
	kind: StorageClass
	metadata:
	  name: managed-nfs-storage
	provisioner: bigdata-provisioner    # or choose another name, must match deployment's env PROVISIONER_NAME'
	parameters:
	  archiveOnDelete: "false"
	```
2. PVC 만들기
	``` yaml
	kind: PersistentVolumeClaim
	apiVersion: v1
	metadata:
	  name: test-claim
	  annotations:
	    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
	spec:
	  accessModes:
	    - ReadWriteMany
	  resources:
	    requests:
	      storage: 1Mi
	```
3. 파드에서 볼륨사용 및 볼륨테스트를 위한 SUCCESS 파일 생성
	``` yaml
	kind: Pod
	apiVersion: v1
	metadata:
	  name: test-pod
	spec:
	  containers:
	  - name: test-pod
	    image: gcr.io/google_containers/busybox:1.24
	    command:
	      - "/bin/sh"
	    args:
	      - "-c"
	      - "touch /mnt/SUCCESS && exit 0 || exit 1"
	    volumeMounts:
	      - name: nfs-pvc
	        mountPath: "/mnt"
	  restartPolicy: "Never"
	  volumes:
	    - name: nfs-pvc
	      persistentVolumeClaim:
	        claimName: test-claim
	```
5.  테스트 - (마운트된 폴더에 SUCCESS 생성여부 체크)
	``` bash
	bigdata@datateam-server5:~/kubernetes/external-storage/nfs-client/deploy$ cd /mnt/kube_nfs/
	bigdata@datateam-server5:/mnt/kube_nfs$ ls
	nfs-test-claim-2-pvc-81ade289-d84c-422e-bad8-652497a474df  nfs-test-claim-pvc-0f45e2f0-7a97-4abb-b1dd-d8988352db21  test.file
	bigdata@datateam-server5:/mnt/kube_nfs$ cd nfs-test-claim-pvc-0f45e2f0-7a97-4abb-b1dd-d8988352db21/
	bigdata@datateam-server5:/mnt/kube_nfs/nfs-test-claim-pvc-0f45e2f0-7a97-4abb-b1dd-d8988352db21$ ls
	SUCCESS
	```


**NFS 마운트 하나당 storageClass 하나를 만들면 됨. 모든 네임스페이스에서 사용가능**
**SA는 네임스페이스마다 만듬. 이걸 통해 PV,PVC 권한 조정가능**

> Written with [StackEdit](https://stackedit.io/).
