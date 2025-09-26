##  Kubernetes 모니터링 환경 구축 (Prometheus + Grafana + NFS 연동)

본 문서는 **NFS 스토리지를 기반으로 Prometheus와 Grafana를 설치**하여 Kubernetes 클러스터 모니터링 환경을 구성하는 과정을 정리한 가이드입니다.  


---

##  1단계: NFS 서버 설정

### NFS 서버에서 공유 디렉토리 설정

```bash
#nfs설정을 노드들이 접근 가능한 IP대역으로 설정
/srv/nfs/kubedata 10.0.2.0/24(rw,sync,no_subtree_check)

ubuntu@nfs-server:~$ sudo cat /etc/exports
/srv/nfs/kubedata 10.0.2.0/24(rw,sync,no_subtree_check)

#exports 설정 적용
ubuntu@nfs-server:~$ sudo exportfs -a

#exports 설정 확인
ubuntu@nfs-server:~$ sudo exportfs -v
/srv/nfs/kubedata
                10.0.2.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```



### 워커 노드에 NFS 클라이언트 설치

```bash

sudo apt-get update
sudo apt-get install nfs-common

```


---

###  2단계:K8s 마스터 노드에서 PV(PersistentVolume) 생성

PV생성을 진행하며, PV를 NFS서버의 경로로 설정.

```bash
ubuntu@masternode:~/k8s-test$ cat pv-nfs.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.0.2.30
    path: "/srv/nfs/kubedata"
ubuntu@masternode:~/k8s-test$
ubuntu@masternode:~/k8s-test$ kubectl apply -f pv-nfs.yaml
persistentvolume/nfs-pv created

#생성한 PV 확인
ubuntu@masternode:~/k8s-test$ kubectl get pv -o wide
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE     VOLUMEMODE
nfs-pv     10Gi       RWX            Retain           Available                                      <unset>                          2m19s   Filesystem

```

**- storage: 10Gi # 용량은 의미상으로만 사용됨 (NFS는 서버 용량을 따름)** <br>
**- ReadWriteMany # NFS의 핵심 장점이며, 여러 파드가 동시에 접근 가능** <br>
**- server: <NFS-Server의 IP_주소>**

---

###  3단계: 헬름 리포지토리 추가 
<br>

**프로메테우스와 그라파나 차트를 다운로드할 수 있는 공식 저장소를 Helm에 추가.**

<br>

```bash
# Prometheus 차트 포지토리 추가
ubuntu@masternode:~$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories

# Grafana 차트 리포지토리 추가
ubuntu@masternode:~$ helm repo add grafana https://grafana.github.io/helm-charts
"grafana" has been added to your repositories

ubuntu@masternode:~$ helm repo list
NAME                    URL
bitnami                 https://charts.bitnami.com/bitnami
prometheus-community    https://prometheus-community.github.io/helm-charts
grafana                 https://grafana.github.io/helm-charts
```
---

<br>

### 🚀 4단계: Prometheus 설치 (NFS 연동)<br>

**헬름 차트를 이용해 프로메테우스를 설치. 여기서는 모니터링 관련 도구들을 monitoring이라는 별도의 네임스페이스에 설치하여 관리함.**

```bash
# 'prometheus'라는 이름으로 prometheus-community/prometheus 차트를 monitoring 네임스페이스에 설치
# 네임스페이스가 없으면 --create-namespace 옵션으로 자동 생성
ubuntu@masternode:~$ helm install prometheus prometheus-community/prometheus --create-namespace --namespace monitoring


ubuntu@masternode:~/k8s-test$ helm install prometheus prometheus-community/prometheus \
  --create-namespace --namespace monitoring \
  --set server.persistentVolume.enabled=true \
  --set server.persistentVolume.storageClass="" \
  --set server.persistentVolume.volumeName=nfs-pv \
  --set server.persistentVolume.size=8Gi \
  --set server.persistentVolume.accessModes[0]=ReadWriteMany \
  --set alertmanager.persistentVolume.enabled=false

```

---

<br>

### 📈 5단계: Grafana 설치<br>

```bash
ubuntu@masternode:~/k8s-test$ helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.existingClaim=prometheus-server


```


<br>

**Grafana 관리자 비밀번호 확인**
```bash

ubuntu@masternode:~/k8s-test$ kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

#포트포워딩
ubuntu@masternode:~/k8s-test$ export GRAFANA_POD_NAME=$(kubectl get pods --namespace monitoring \
  -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" \
  -o jsonpath="{.items[0].metadata.name}")

ubuntu@masternode:~/k8s-test$
ubuntu@masternode:~/k8s-test$ kubectl --namespace monitoring port-forward $GRAFANA_POD_NAME 3000

```
<br>

**✅ Pod 상태 확인**
<br>

```bash
ubuntu@masternode:~$ kubectl get pods -o wide -n monitoring
NAME                                                 READY   STATUS    RESTARTS        AGE     IP                NODE           NOMINATED NODE   READINESS GATES
grafana-549c7bcc9f-hpx7j                             1/1     Running   0               4h44m   192.168.52.152    workernode01   <none>           <none>
prometheus-alertmanager-0                            0/1     Pending   0               91m     <none>            <none>         <none>           <none>
prometheus-kube-state-metrics-7b845c4b4d-jx2z8       1/1     Running   0               4h46m   192.168.246.208   workernode02   <none>           <none>
prometheus-prometheus-node-exporter-cjb8l            1/1     Running   0               4h46m   10.0.2.25         workernode02   <none>           <none>
prometheus-prometheus-node-exporter-g7pmq            1/1     Running   0               4h46m   10.0.2.20         workernode01   <none>           <none>
prometheus-prometheus-node-exporter-gd2hv            1/1     Running   0               4h46m   10.0.2.15         masternode     <none>           <none>
prometheus-prometheus-pushgateway-65966498f7-94lf9   1/1     Running   0               4h46m   192.168.52.151    workernode01   <none>           <none>
prometheus-server-576cf97f7-5nsvr                    2/2     Running   23 (108m ago)   4h46m   192.168.246.209   workernode02   <none>           <none>

```
<br>

---

###  Grafana 접속

브라우저에서 http://localhost:3000
 접속 후 Grafana 로그인
(초기 계정: admin / <위에서 확인한 비밀번호>)  <br>

- 로그인 화면 예시 <br>
<img width="1913" height="963" alt="Grafana 로그인화면" src="https://github.com/user-attachments/assets/5bce23bf-cbee-4e2a-a873-45ff404f7a26" />


<br>

### Prometheus 연동 후 대시보드 예시  <br>

**MasterNode 모니터링 대시보드**
<img width="1876" height="950" alt="Grafana dashboard" src="https://github.com/user-attachments/assets/ee3c83c2-553a-458c-a135-3dc53dafd4e3" />

<br>

**WorkerNode01 모니터링 대시보드**
<img width="1893" height="897" alt="Grafana worker01대쉬보드" src="https://github.com/user-attachments/assets/66b9fe69-5e88-4a3b-9332-1505dfde2203" />

<br>

**WorkerNode02 모니터링 대시보드**
<img width="1887" height="945" alt="Grafana worker02 대쉬보드" src="https://github.com/user-attachments/assets/16c9392a-d5dd-4549-b683-c021fab2b972" />



---


###  트러블 슈팅

<br>

**Helm으로 Grafana 설치시, 아래와 같이 에러가 발생하여서 Grafana pod가 제대로 구동하지 않음.** <br>

```bash

ubuntu@masternode:~/k8s-test$ kubectl get pods -o wide -n monitoring
NAME                                                 READY   STATUS                  RESTARTS         AGE   IP                NODE           NOMINATED NODE   READINESS GATES
grafana-549c7bcc9f-hpx7j                             0/1     Init:CrashLoopBackOff   12 (3m56s ago)   40m   192.168.52.152    workernode01   <none>           <none>
prometheus-alertmanager-0                            0/1     Pending                 0                43m   <none>            <none>         <none>           <none>
prometheus-kube-state-metrics-7b845c4b4d-jx2z8       1/1     Running                 0                43m   192.168.246.208   workernode02   <none>           <none>
prometheus-prometheus-node-exporter-cjb8l            1/1     Running                 0                43m   10.0.2.25         workernode02   <none>           <none>
prometheus-prometheus-node-exporter-g7pmq            1/1     Running                 0                43m   10.0.2.20         workernode01   <none>           <none>
prometheus-prometheus-node-exporter-gd2hv            1/1     Running                 0                43m   10.0.2.15         masternode     <none>           <none>
prometheus-prometheus-pushgateway-65966498f7-94lf9   1/1     Running                 0                43m   192.168.52.151    workernode01   <none>           <none>
prometheus-server-576cf97f7-5nsvr                    2/2     Running                 0                43m   192.168.246.209   workernode02   <none>           <none>
ubuntu@masternode:~/k8s-test$
```

<br>

**describe로 확인한 결과, nfs 저장소를 사용하기 위해 시작 단계에서 권한 문제가 발생 하였음.**

<br>

**NFS서버가 루트 권한을 얻은 노드들을 믿지 못하여서 발생한 문제였고,** <br> 
**해당 문제를 해결하기 위해 NFS 서버의 Root 권한 문제 설정 진행**
```bash
ubuntu@masternode:~/k8s-test$ kubectl describe pods grafana-549c7bcc9f-hpx7j -n monitoring

Init Containers:
  init-chown-data:
    Container ID:    containerd://af86a1a2610fb677a045c3cebc6fd25240ae0b8d255d6015aed6f4f476b50133
    Image:           docker.io/library/busybox:1.31.1
    Image ID:        docker.io/library/busybox@sha256:95cf004f559831017cdf4628aaf1bb30133677be8702a8c5f2994629f637a209
    Port:            <none>
    Host Port:       <none>
    SeccompProfile:  RuntimeDefault
    Command:
      chown
      -R
      472:472
      /var/lib/grafana
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
      Started:      Fri, 26 Sep 2025 03:04:35 +0000
      Finished:     Fri, 26 Sep 2025 03:04:35 +0000
    Ready:          False
    Restart Count:  11
    Environment:    <none>
    Mounts:
      /var/lib/grafana from storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-jv79c (ro)


```

<br>

**NFS root권한 설정 진행**

```bash
/srv/nfs/kubedata 10.0.2.0/24(rw,sync,no_subtree_check,no_root_squash)

#설정 변경 적용
ubuntu@nfs-server:~$ sudo exportfs -a
ubuntu@nfs-server:~$
#설정 상태 확인
ubuntu@nfs-server:~$ sudo exportfs -v
/srv/nfs/kubedata
                10.0.2.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)

```
<br>

---

**Grafana pod 상태 확인**

```bash
ubuntu@masternode:~/k8s-test$ kubectl get pods -o wide -n monitoring
NAME                                                 READY   STATUS    RESTARTS   AGE   IP                NODE           NOMINATED NODE   READINESS GATES
grafana-549c7bcc9f-hpx7j                             1/1     Running   0          43m   192.168.52.152    workernode01   <none>           <none>
prometheus-kube-state-metrics-7b845c4b4d-jx2z8       1/1     Running   0          45m   192.168.246.208   workernode02   <none>           <none>
prometheus-prometheus-node-exporter-cjb8l            1/1     Running   0          45m   10.0.2.25         workernode02   <none>           <none>
prometheus-prometheus-node-exporter-g7pmq            1/1     Running   0          45m   10.0.2.20         workernode01   <none>           <none>
prometheus-prometheus-node-exporter-gd2hv            1/1     Running   0          45m   10.0.2.15         masternode     <none>           <none>
prometheus-prometheus-pushgateway-65966498f7-94lf9   1/1     Running   0          45m   192.168.52.151    workernode01   <none>           <none>
prometheus-server-576cf97f7-5nsvr                    2/2     Running   0          45m   192.168.246.209   workernode02   <none>           <none>


```
<br>



###  최종 결과

**Prometheus + Grafana 설치 완료** <br>

**NFS PersistentVolume 연동 성공** <br>

**Kubernetes 클러스터 모니터링 대시보드 활성화** <br>

