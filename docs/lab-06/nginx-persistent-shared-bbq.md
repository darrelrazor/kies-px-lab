# Lab 06 후보. NGINX 영구·공유 볼륨과 Portworx BBQ

기존 LAB 05 리소스와 겹치지 않도록 이 문서에서는 `px-nginx-lab`과 `portworx-bbq` 네임스페이스만 사용합니다.

### Task 1. NGINX 영구 볼륨 테스트

1. 실습용 네임스페이스를 생성합니다.

```bash
kubectl create namespace px-nginx-lab \
  --dry-run=client -o yaml | kubectl apply -f -
```

2. PVC와 NGINX Pod를 생성합니다. `index.html`은 Portworx PVC에 저장됩니다.

```bash
cat <<'EOF' > ~/nginx-persistent.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-content-rwo
  namespace: px-nginx-lab
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: px-csi-db
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-persistent
  namespace: px-nginx-lab
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
  volumes:
    - name: content
      persistentVolumeClaim:
        claimName: nginx-content-rwo
EOF

kubectl apply -f ~/nginx-persistent.yaml
kubectl get pvc,pod -n px-nginx-lab
```

3. `index.html`을 생성하고 웹에서 확인합니다.

```bash
kubectl exec -n px-nginx-lab nginx-persistent -- \
  rm -f /usr/share/nginx/html/index.html

kubectl exec -i -n px-nginx-lab nginx-persistent -- \
  sh -c 'cat > /usr/share/nginx/html/index.html' <<'EOF'
<h1>수정된 index.html</h1>
<p>Pod를 다시 만들어도 이 내용은 유지됩니다.</p>
EOF

kubectl port-forward -n px-nginx-lab pod/nginx-persistent 8080:80
```

다른 터미널이나 브라우저에서 `http://localhost:8080`에 접속합니다.

4. Pod를 삭제하고 같은 YAML로 다시 생성합니다.

```bash
kubectl delete pod -n px-nginx-lab nginx-persistent
kubectl apply -f ~/nginx-persistent.yaml
kubectl wait -n px-nginx-lab --for=condition=Ready \
  pod/nginx-persistent --timeout=180s
kubectl exec -n px-nginx-lab nginx-persistent -- \
  cat /usr/share/nginx/html/index.html
```

수정한 내용이 출력되면 Pod를 다시 만들어도 PVC 데이터가 유지된 것입니다.

### Task 2. NGINX 공유 볼륨 테스트

1. 모든 워커 노드에서 Sharedv4에 필요한 NFS 패키지를 준비합니다.

```bash
dnf install -y nfs-utils
systemctl enable --now rpcbind
```

2. Sharedv4 PVC와 NGINX Pod 3개를 생성합니다. 공유 파일은 `/shared`, 웹 파일은 각 Pod 내부에 둡니다.

```bash
cat <<'EOF' > ~/nginx-shared.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: px-nginx-rwx
provisioner: pxd.portworx.com
parameters:
  repl: "2"
  sharedv4: "true"
  sharedv4_svc_type: "ClusterIP"
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-content-rwx
  namespace: px-nginx-lab
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: px-nginx-rwx
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-shared
  namespace: px-nginx-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-shared
  template:
    metadata:
      labels:
        app: nginx-shared
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          command: ["/bin/sh", "-c"]
          args:
            - |
              [ -f /shared/message.txt ] || \
                echo '3개 Pod가 같은 Portworx RWX PVC를 사용합니다.' \
                > /shared/message.txt
              printf '<h1>NGINX Shared Volume</h1>\n<p>응답 Pod: %s</p>\n' \
                "$HOSTNAME" > /usr/share/nginx/html/index.html
              exec nginx -g 'daemon off;'
          ports:
            - containerPort: 80
          volumeMounts:
            - name: content
              mountPath: /shared
      volumes:
        - name: content
          persistentVolumeClaim:
            claimName: nginx-content-rwx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-shared
  namespace: px-nginx-lab
spec:
  selector:
    app: nginx-shared
  ports:
    - port: 80
      targetPort: 80
EOF

kubectl apply -f ~/nginx-shared.yaml
kubectl get pvc,pod,svc -n px-nginx-lab -o wide
```

3. 첫 번째 Pod에서 공유 파일을 생성하고 모든 Pod에서 읽습니다.

```bash
WRITER=$(kubectl get pod -n px-nginx-lab -l app=nginx-shared \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n px-nginx-lab "$WRITER" -- \
  sh -c 'date > /shared/shared.txt'

for POD in $(kubectl get pod -n px-nginx-lab -l app=nginx-shared \
  -o jsonpath='{.items[*].metadata.name}'); do
  echo "[$POD]"
  kubectl exec -n px-nginx-lab "$POD" -- \
    cat /shared/shared.txt
done
```

모든 Pod에서 같은 내용이 출력되면 RWX PVC를 정상적으로 공유하고 있는 것입니다.

4. Service를 로컬 포트로 연결합니다.

```bash
kubectl port-forward -n px-nginx-lab service/nginx-shared 8081:80
```

다른 터미널에서 아래 명령을 실행하거나 브라우저에서 `http://localhost:8081`을 여러 번 새로고침합니다.

```bash
for i in {1..9}; do
  curl -s http://localhost:8081 \
    | grep -o 'nginx-shared-[^<]*'
done
```

Pod 이름이 바뀌면 Service가 세 NGINX Pod로 요청을 분산하고 있는 것입니다.

### Task 3. Portworx BBQ 설치

1. Helm과 Portworx CSI StorageClass를 확인합니다.

```bash
helm version
kubectl get sc px-csi-db
```

2. SUSE Helm 저장소를 추가합니다.

```bash
helm repo add suse-lab-setup https://opensource.suse.com/lab-setup
helm repo update
helm search repo suse-lab-setup/portworx-bbq
```

3. BBQ 설정 파일을 생성합니다.

```bash
cat <<'EOF' > ~/px-bbq-values.yaml
front:
  name: pxbbq-web
  image: eshanks16/pxbbq
  tag: v2
  replicaCount: 3
  containerPort: 8080
  healthEndpoint: /healthz
  host: ""
  port: 80
  db:
    host: portworx-bbq-mongodb
    port: "27017"
    username: root
    userpwd: root-password
    tls: ""

ingress:
  enabled: false

mongodb:
  enabled: true
  auth:
    rootPassword: root-password
    username: porxie
    password: porxie-password
    database: admin
  persistence:
    enabled: true
    storageClass: px-csi-db
    size: 8Gi
  image:
    registry: docker.io
    repository: bitnamilegacy/mongodb
    tag: 7.0.14-debian-12-r3
EOF
```

4. Portworx BBQ를 설치합니다.

```bash
helm upgrade --install portworx-bbq \
  suse-lab-setup/portworx-bbq \
  --namespace portworx-bbq \
  --create-namespace \
  -f ~/px-bbq-values.yaml
```

5. 애플리케이션과 MongoDB PVC 상태를 확인합니다.

```bash
helm status portworx-bbq -n portworx-bbq
kubectl get pod,svc,pvc -n portworx-bbq
kubectl get pvc -n portworx-bbq \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,SC:.spec.storageClassName,VOLUME:.spec.volumeName
```

6. 기존 NGINX Gateway에 BBQ용 HTTPRoute를 추가합니다.

```bash
cat <<'EOF' > ~/px-bbq-route.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: px-bbq
  namespace: portworx-bbq
spec:
  parentRefs:
    - name: nginx-gateway
      namespace: nginx-gateway
      sectionName: http
  hostnames:
    - px-bbq.xx.px.lab
  rules:
    - backendRefs:
        - name: pxbbq-web
          port: 80
EOF

vi ~/px-bbq-route.yaml
# xx를 자신의 LAB 번호로 변경한 뒤 저장합니다.
kubectl apply -f ~/px-bbq-route.yaml
kubectl get httproute -n portworx-bbq
```

7. 웹 브라우저에서 BBQ에 접속하여 데이터를 생성합니다.

```text
http://px-bbq.xx.px.lab
```

8. MongoDB Pod를 재시작하고 생성한 데이터가 유지되는지 확인합니다.

```bash
kubectl rollout restart statefulset/portworx-bbq-mongodb \
  -n portworx-bbq
kubectl rollout status statefulset/portworx-bbq-mongodb \
  -n portworx-bbq --timeout=300s
```

웹 페이지를 새로고침했을 때 기존 데이터가 남아 있으면 MongoDB가 Portworx PVC를 정상적으로 사용하고 있는 것입니다.

### 선택 사항. 실습 리소스 정리

> Warning: 아래 명령어는 이 문서에서 생성한 Pod와 PVC를 삭제합니다. `reclaimPolicy: Delete`인 볼륨도 함께 삭제됩니다.

```bash
helm uninstall portworx-bbq -n portworx-bbq
kubectl delete namespace portworx-bbq

kubectl delete namespace px-nginx-lab
kubectl delete storageclass px-nginx-rwx
```

## 참고 자료

- [Portworx CSI StorageClass](https://docs.portworx.com/portworx-enterprise/reference/storageclass)
- [Portworx Sharedv4 Volumes](https://docs.portworx.com/portworx-enterprise/concepts/shared-volumes)
- [SUSE Portworx BBQ Helm Chart](https://opensource.suse.com/lab-setup/)

---

[처음으로](../../README.md) | [기존 LAB 06](./dynamic-volume-provisioning.md)
