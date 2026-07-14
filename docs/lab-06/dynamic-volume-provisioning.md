# Lab 06. Dynamic Volume Provisioning

### Task 1. Portworx StorageClass 생성

1. StorageClass YAML 파일을 생성합니다.

```bash
vi ~/px-postgres-sc.yaml
```

```yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: px-postgres
provisioner: kubernetes.io/portworx-volume
parameters:
  io_profile: db_remote
  repl: "3"
  shared: "true"
```

2. StorageClass를 생성합니다.

```bash
kubectl apply -f ~/px-postgres-sc.yaml
```

3. StorageClass 목록을 확인합니다.

```bash
kubectl get sc px-postgres -n portworx
```

4. StorageClass 상세 정보를 확인합니다.

```bash
kubectl describe sc px-postgres
```

### Task 2. PVC 생성

1. `postgres` 네임스페이스를 생성합니다.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: postgres
EOF
```

2. PVC YAML 파일을 생성합니다.

```bash
vi ~/postgres-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  annotations:
    volume.beta.kubernetes.io/storage-class: px-postgres
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
```

3. PVC를 생성합니다.

```bash
kubectl apply -f ~/postgres-pvc.yaml --namespace=postgres
```

4. PVC 목록을 확인합니다.

```bash
kubectl get pvc postgres-data -n postgres
```

5. PVC 상세 정보를 확인합니다.

```bash
kubectl describe pvc postgres-data -n postgres
```

6. PVC 상세 정보에서 StorageClass가 Task 1에서 생성한 `px-postgres`이며, Portworx 프로비저너가 볼륨을 생성하고 PVC를 바인딩한 것을 확인합니다.
7. PVC 상세 정보에서 볼륨명을 복사합니다.
8. 복사한 PVC 볼륨명으로 Portworx에 볼륨이 동적으로 생성되었는지 확인합니다.

```bash
pxctl1 volume inspect <pvc-volume-name>
```

9. Task 1에서 생성한 StorageClass 명세와 동적으로 생성된 볼륨의 명세를 비교합니다.

```bash
kubectl describe sc px-postgres
```

> Note: 현재 볼륨은 로컬 스토리지 풀에 등록되지 않은 `detached` 상태입니다. 다음 Task에서 해당 볼륨에 바인드된 PVC로 앱 배포가 완료되면, 해당 Pod가 배포된 노드의 로컬 스토리지 풀로 Attach가 자동으로 이루어집니다.

### Task 3. PostgreSQL 배포

1. PostgreSQL 배포를 위한 Deployment YAML 파일을 생성합니다.

```bash
vi ~/postgres-app.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      schedulerName: stork
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: px/enabled
                    operator: NotIn
                    values:
                      - "false"
      containers:
        - name: postgres
          image: postgres:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              value: pguser
            - name: POSTGRES_PASSWORD
              value: pguser
            - name: PGBENCH_PASSWORD
              value: pguser
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgredb
      volumes:
        - name: postgredb
          persistentVolumeClaim:
            claimName: postgres-data
```

2. PostgreSQL 애플리케이션을 배포합니다.

```bash
kubectl apply -f ~/postgres-app.yaml --namespace=postgres
```

3. PostgreSQL 리소스 배포 상태를 확인합니다.

```bash
kubectl get all -n postgres
```

4. Portworx에서 볼륨 정보를 다시 조회합니다.

```bash
pxctl1 volume inspect <pvc-volume-name>
```

> Note: PostgreSQL 애플리케이션 배포 전과 달리 해당 볼륨은 PostgreSQL이 배포된 노드의 로컬 스토리지 풀로 Attach됩니다. 또한 `Volume consumers` 정보가 추가되어 해당 볼륨을 어떤 애플리케이션에서 사용하는지 확인할 수 있습니다.

### Task 4. 볼륨 공유용 Pod 추가 배포

1. 테스트 Pod 배포를 위한 YAML 파일을 생성합니다.

> Note: 이전 단계의 PostgreSQL이 배포된 노드와 다른 노드에 배포하기 위해 `nodeSelector`를 지정합니다. 볼륨을 공유하기 위해 이전 단계에서 생성한 Portworx용 PVC를 지정합니다.

```bash
vi ~/test-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/test-webserver
      volumeMounts:
        - name: shared-volume
          mountPath: /shared-portworx-volume
  nodeSelector:
    kubernetes.io/hostname: "w3.px.lab"
  volumes:
    - name: shared-volume
      persistentVolumeClaim:
        claimName: postgres-data
```

2. 테스트 Pod를 배포합니다.

```bash
kubectl apply -f ~/test-pod.yaml --namespace=postgres
```

3. Pod가 `Running` 상태인지 확인합니다.

```bash
kubectl get all -n postgres
```

4. Portworx 볼륨 정보를 조회합니다.

```bash
pxctl1 volume inspect <pvc-volume-name>
```

> Note: Portworx 볼륨 정보의 `Volume consumers`에 테스트용 Pod가 등록됩니다. 해당 볼륨은 워커 노드 1번에 Attach되었지만 공유 볼륨이므로 다른 노드에 배포된 Pod도 접근할 수 있습니다.

---

[처음으로](../../README.md) | [이전 LAB](../lab-05/portworx-operations.md)
