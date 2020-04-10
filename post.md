# 노드의 할당 가능한 리소스(allocatable resource) 변경

노드마다 쿠버네티스에서 사용가능 한 리소스가 할당(allocatable resource)되어있다. 공식 웹사이트에서는 리소스의 할당량을 임의로 조절하기보단, 파드의 필요한 리소스(request)와 최대 리소스(limit) 을 지정하라고 되어있지만 임의로 조정할 때가 생길 수도 있다. 이 글에서는 노드의 할당된 리소스를 어떻게 변경하는지 알아보자


## 노드의 사용가능한 리소스보기
``` bash
$kubectl get nodes                           
NAME               STATUS   ROLES    AGE   VERSION
datateam-server3   Ready    <none>   44d   v1.17.3
datateam-server4   Ready    <none>   43d   v1.17.3
datateam-server5   Ready    master   44d   v1.17.3

$kubectl get nodes datateam-server3  -o=jsonpath='{"capacity:    "}{.status.capacity}{"\n"}{"allocatable: "}{.status.allocatable}'
capacity:    map[cpu:8 ephemeral-storage:490215384Ki hugepages-1Gi:0 hugepages-2Mi:0 memory:16299064Ki pods:110]
allocatable: map[cpu:8 ephemeral-storage:451782497147 hugepages-1Gi:0 hugepages-2Mi:0 memory:16196664Ki pods:110]%         
```
.status.capacity : 노드의 총 자원의 양 (cpu: 8core, ram: 약 16G, gpu: 1unit, storage: 약 959G)
.status.allocatable: 쿠버네티스에서 사용 가능한 자원의 양 (cpu: 8core, ram: 약 16G, gpu: 1unit, storage: 약 884G)

## allocatable 계산방법

allocatble = capacity - (kube reserved + system reserved + eviction-threshold)

capacity: 전체 자원
kube reserved: Kubernetes가 예약한 자원
system reserved: system이 예약한 자원
eviction-threshold:  노드의 최소 자원 임계값(노드의 리소스가 지정값보다 낮아지면 파드를 제거하거나 다른 노드로 옮김)

## allocatable값 변경하기 

** kubectl 플래그 변경 (KubectlConfiguration 변경)**
1.  KubectlConfiguration 파일 경로 얻기	 
	\- kubeadm
	```
	bigdata@datateam-server5:~/ai-serv-yaml/asr-svc$ sudo cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf  
	 # Note: This dropin only works with kubeadm and kubelet v1.11+  
	 [Service]
	 
	 	(생략)
	   
	 Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"  
	 # This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
	 
	    (생략)
	```
	\-kubespray
	```
	imgamggi@kubespray-master:~$ sudo cat /etc/systemd/system/kubelet.service 
	   
	EnvironmentFile=-/etc/kubernetes/kubelet.env

	
	imgamggi@kubespray-master:~$ sudo cat /etc/kubernetes/kubelet.env

	  (생략)
	  
	KUBELET_ARGS="--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \  
	--config=/etc/kubernetes/kubelet-config.yaml  \  
	--kubeconfig=/etc/kubernetes/kubelet.conf \  
	--pod-infra-container-image=[k8s.gcr.io/pause:3.1](http://k8s.gcr.io/pause:3.1)  \  
	--runtime-cgroups=/systemd/system.slice \
	"
	  (생략)
	```
	
	
2. 	config.yaml 파일 스펙에 맞게 수정 (스펙 정보: [https://pkg.go.dev/k8s.io/kubelet/config/v1beta1?tab=doc#KubeletConfiguration](https://pkg.go.dev/k8s.io/kubelet/config/v1beta1?tab=doc#KubeletConfiguration))
	 ```
	 apiVersion: [kubelet.config.k8s.io/v1beta1](http://kubelet.config.k8s.io/v1beta1)  
	 kind: KubeletConfiguration  
	 ...     (생략)  
	 evictionHard:  
	 memory.available: 100Mi  
	 kubeReserved:  
	 cpu: 100m  
	 memory: 100Mi  
	 systemReserved:  
	 cpu: 100m  
	 memory: 100Mi
	 ```


3.  kubelet 서비스 변경사항 적용
	 ```
	 sudo systectl restart kubelet
	 ```


> Written with [StackEdit](https://stackedit.io/).
