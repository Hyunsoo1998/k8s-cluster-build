
# Kubernetes Persistent Storage 실습: PV, PVC, Pod 연결하기
이 리포지토리는 Kubernetes 환경에서 **PersistentVolume (PV)**, **PersistentVolumeClaim (PVC)**, 그리고 **Pod**를 연결하는 과정을 정리한 예제입니다.  
Pod가 영속 스토리지를 사용할 수 있도록 설정하는 방법을 실습하였습니다.


<br>

## 🗂️ 구성 요소

### 1. PersistentVolume 생성 (PV)

`pv-test.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hskim-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteOnce
  storageClassName: hskim
  hostPath:
    path: "/mnt/data"
```

- 호스트 노드의 /mnt/data 경로를 PV로 설정.

- 500Mi 스토리지 용량 할당.


### 2. PersistentVolumeClaim (PVC)

`pvc-test.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hskim-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 250Mi
  storageClassName: hskim
```

- PVC는 PV, Pod와 매핑 됨.
- RWO: 하나의 노드에서만 해당 스토리지를 읽고 쓸 수 있도록 제한.
- hskim-pv와 바인딩됨.

- 250Mi 용량 요청.

### 3. Pod (Nginx)

`hskim-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: nginx-storage
      persistentVolumeClaim:
        claimName: hskim-pvc
```

- Nginx 컨테이너를 실행.

- /usr/share/nginx/html 경로에 PVC 마운트.

### 🔍 실행 결과

**리소스 생성**
```bash
kubectl apply -f pv-test.yaml
kubectl apply -f pvc-test.yaml
kubectl apply -f hskim-pod.yaml

```

**상태 확인**
```bash
ubuntu@masternode:~$ kubectl get pv -o wide
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE    VOLUMEMODE
hskim-pv   500Mi      RWO            Retain           Bound    default/hskim-pvc   hskim          <unset>                          142m   Filesystem

ubuntu@masternode:~$ kubectl get pvc -o wide
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE    VOLUMEMODE
hskim-pvc   Bound    hskim-pv   500Mi      RWO            hskim          <unset>                 137m   Filesystem

ubuntu@masternode:~$ kubectl get pods -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
nginx-pod   1/1     Running   0          41m   192.168.52.146   workernode01   <none>           <none>

```

---


**pod 내부에 직접 접속하여 파일 생성 및 바인드 테스트**

```bash
kubectl exec -it nginx-pod -- /bin/bash

root@nginx-pod:/usr/share/nginx/html# cat test.html
PV-PVC mapping test
```

---

**노드의 실제 디렉토리 확인**

```bash
ubuntu@workernode01:~$ cat /mnt/data/test.html
PV-PVC mapping test
```

---
<br>

### ✅ Pod에서 작성한 파일이 호스트 노드의 /mnt/data 경로에 저장되어 있는 것을 확인할 수 있다.

### 📌결론

Pod는 직접 PV에 접근할 수 없고 PVC를 통해서만 PV와 연결할 수 있다.

PVC가 바인딩되면 Pod는 컨테이너 내부 경로(/usr/share/nginx/html)에서 데이터를 영속적으로 저장할 수 있다.

PV-PVC-Pod 매핑 과정을 통해 Kubernetes의 영속 스토리지 개념을 실습하였다.

### 📜 참고
https://kubernetes.io/ko/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims
