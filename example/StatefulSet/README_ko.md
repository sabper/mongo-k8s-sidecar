# Running a MongoDB Database in Kubernetes with StatefulSets

이 문서는 quiklabs `GSP022 Running a MongoDB Database in Kubernetes with StatefulSets` 의 내용을 실습하면서 작성 되었습니다.

> qwiklabs <https://qwiklabs.com/focuses/6909>

> blog <http://blog.kubernetes.io/2017/01/running-mongodb-on-kubernetes-with-statefulsets.html>

<br>

## 사전 작업

### Mac OS X 에서 kuberctl 사용 - gcloud shell 대신 local shell 사용

> GCP kubernetes Quickstart <https://cloud.google.com/kubernetes-engine/docs/quickstart?hl=ko>

* Google Cloud SDK 설치
  > Quickstart for Mac OS X <https://cloud.google.com/sdk/docs/quickstart-mac-os-x?hl=ko>

* kubectl 설치
  > Kubernetes Quickstat <https://cloud.google.com/kubernetes-engine/docs/quickstart?hl=ko>

  ```shell
  gcloud components install kubectl
  ```

* Configuring default settings for gcloud

  * 새로 생성한 gcp project 로 gcloud default project 세팅

  ```shell
  gcloud config set project gcp-study

  ```

  * compute/zone 설정

  ```shell
  gcloud config set compute/zone us-west1-a
  ```

<br>

### GCP kubnetes cluster 생성

* 새로운 Kubernetes cluster 생성 - 소요 시간 약 5분

  * vm instance 3대 구성됨

  * vm instance 개수 등을 설정 옵션
    > <https://cloud.google.com/sdk/gcloud/reference/container/clusters/create>

  ```shell
  gcloud container clusters create hello-mongo
  ```

  ```shell
  Creating cluster hello-mongo...done.
  Created [https://container.googleapis.com/v1/projects/gcp-stupdy/zones/us-west1-a/clusters/hello-mongo].
  kubeconfig entry generated for hello-mongo.
  NAME         ZONE        MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION  NUM_NODES  STATUS
  hello-mongo  us-west1-a  1.7.8-gke.0     35.197.87.230  n1-standard-1  1.7.8-gke.0   3          RUNNING
  ```

* 새로 만든 kubernetes 클러스터 인증
  ```shell
  gcloud container clusters get-credentials hello-mongo
  ```

<br>
<br>

## Lab

### Setting up the Storage Class

* 스토리지 클래스 생성 - 데이터베이스가 사용할 스토리지 ssd, hdd
  > storate class <https://kubernetes.io/docs/concepts/storage/storage-classes/>

  > GCP storate option <https://cloud.google.com/compute/docs/disks/>

* storage 설정 확인

  ```shell
  cat googlecloud_ssd.yaml
  ```

  ```shell
  kind: StorageClass
  apiVersion: storage.k8s.io/v1beta1
  metadata:
    name: fast
  provisioner: kubernetes.io/gce-pd
  parameters:
    type: pd-ssd
  ```

* storage 생성

  ```shell
  kubectl apply -f googlecloud_ssd.yaml
  ```

<br>

### Deploying the Headless Service and StatefulSet

* Headless Service
  
  * service 설정에서 cluster ip 를 none 으로 세팅
  
  * 외부 ip가 존재하지 않고 자체 dns가 구성됨
  
  > <https://kubernetes.io/docs/concepts/services-networking/service/#headless-services>

* StatefulSet

  * mongodb replica set pod 을 구성하는 service

  > StatefulSet <https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/>

  > Deployments <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>

* headless, statatefulset 으로 mongodb replica pod 구축

  ```shell
  kubectl apply -f mongo-statefulset.yaml
  ```

<br>

### Connect to the MongoDB Replica Set

* Mongodb replica set 로딩 시까지 대기

  statefulset 구성 확인

  ```shell
  kubectl get statefulset
  ```

  DESIRED 와 CURRENT 가 같아야 모든 pod 이 구동된 것임
  ```shell
  NAME      DESIRED   CURRENT   AGE
  mongo     3         3         1m
  ```

  구동된 각 pod 조회
  ```shell
  kubectl get pods
  ```

  ```shell
  NAME      READY     STATUS    RESTARTS   AGE
  mongo-0   2/2       Running   0          3m
  mongo-1   0/2       Pending   0          2m
  ```

  ```shell
  NAME                    READY     STATUS    RESTARTS   AGE
  mongo-0                 2/2       Running   0          2m
  mongo-1                 2/2       Running   0          1m
  mongo-2                 2/2       Running   0          1m
  ```

<br>

### Mongo replica set 확인

* 첫번째 pod 접속 - mongo replica set 중 첫번째

  ```shell
  kubectl exec -ti mongo-0 mongo
  ```

* mongodb 상태 확인

  ```shell
  rs.status()

  or

  rs.conf()
  ```

* 다른 확인 방법 - kubnetes dns 이용

  > <https://kubernetes.io/docs/concepts/services-networking/connect-applications-service>

  ```shell
  $ kubectl get services kube-dns --namespace=kube-system
  NAME       CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
  kube-dns   10.0.0.10    <none>        53/UDP,53/TCP   8m

  $ kubectl run curl --image=radial/busyboxplus:curl -i --tty
  If you don't see a command prompt, try pressing enter.

  [ root@curl-2716574283-n4fh5:/ ]$ nslookup mongo-0.mongo
  Server:    10.31.240.10
  Address 1: 10.31.240.10 kube-dns.kube-system.svc.cluster.local

  Name:      mongo-0.mongo
  Address 1: 10.28.0.6 mongo-0.mongo.default.svc.cluster.local

  $ kubectl attach curl-2716574283-n4fh5 -c curl -i -t
  ```

<br>

### Kubernetes cluster 에 구성된 mongodb replica 초기화

* statefulset 삭제

  ```shell
  $ kubectl delete statefulset mongo

  statefulset "mongo" deleted
  ```

* service 삭제

  ```shell
  $ kubectl delete svc mongo

  service "mongo" deleted
  ```

* volume 삭제

  ```shell
  $ kubectl delete pvc -l role=mongo

  persistentvolumeclaim "mongo-persistent-storage-mongo-0" deleted
  persistentvolumeclaim "mongo-persistent-storage-mongo-1" deleted
  persistentvolumeclaim "mongo-persistent-storage-mongo-2" deleted
  ```

* cluster 삭제 - GCP compute engine instance, instance group, instance template, disk 같이 삭제됨

  ```shell
  $ gcloud container clusters delete "{Kubernetes cluster name}"

  The following clusters will be deleted.
  - [hello-mongo] in [us-central1-a]

  Do you want to continue (Y/n)?  Y

  Deleting cluster hello-mongo...-
  ```

<br>

### 외부에서 kubernetes cluster 내부 mongo replica set 접속 시 참고

> <https://kubernetes.io/docs/concepts/services-networking/ingress/>

<br>
<br>

## Troubleshooting

### kubectl 수행 시 default 인증 요구 에러

* 에러

  ```shell
  error: google: could not find default credentials. See https://developers.google.com/accounts/docs/application-default-credentials for more information.
  ```

* 해결 - default 인증 설정

  > <https://developers.google.com/identity/protocols/application-default-credentials>

  ```shell
  gcloud auth application-default login
  ```

<br>

### PersistentVolumeClaim is not bound

* `https://github.com/thesandlord/mongo-k8s-sidecar.git` 의 설정파일로 구성 시 pod 구동 시 에러 발생

* 재현 시도 하지 않았음
