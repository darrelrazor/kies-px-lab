# Lab 01. 쿠버네티스 클러스터 구성

### Task 1. Lab 환경 확인

1. 쿠버네티스 클러스터를 구성할 대상 노드와 IP 정보를 확인합니다.

| Role | Hostname | IP | CPU(Core) | Memory(GB) | Disks(GB) | OS Version | User |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 마스터 노드 | px-lab-xx-m01 | 192.168.102.xxx | 4 | 6 | 50 | Rocky 8.10 | root |
| 워커 노드 #1 | px-lab-xx-w01 | 192.168.102.xxx | 4 | 8 | 50/100 | Rocky 8.10 | root |
| 워커 노드 #2 | px-lab-xx-w02 | 192.168.102.xxx | 4 | 8 | 50/100 | Rocky 8.10 | root |
| 워커 노드 #3 | px-lab-xx-w03 | 192.168.102.xxx | 4 | 8 | 50/100 | Rocky 8.10 | root |

### Task 2. 노드 설정

1. 모든 노드에서 호스트명을 지정합니다.

```bash
hostnamectl set-hostname <호스트명>
```

| Role | Hostname |
| --- | --- |
| 마스터 노드 | px-lab-xx-m01 |
| 워커 노드 #1 | px-lab-xx-w01 |
| 워커 노드 #2 | px-lab-xx-w02 |
| 워커 노드 #3 | px-lab-xx-w03 |

2. 모든 노드의 `/etc/hosts` 파일에 노드 정보를 동일하게 등록합니다.

```bash
vi /etc/hosts
```

```text
192.168.102.xx  px-lab-xx-m01
192.168.102.xx  px-lab-xx-w01
192.168.102.xx  px-lab-xx-w02
192.168.102.xx  px-lab-xx-w03
```

| Hostname | IP Address |
| --- | --- |
| px-lab-xx-m01 | 192.168.102.xx |
| px-lab-xx-w01 | 192.168.102.xx |
| px-lab-xx-w02 | 192.168.102.xx |
| px-lab-xx-w03 | 192.168.102.xx |

3. 호스트명으로 통신이 되는지 확인합니다.

```bash
ping px-lab-xx-m01
```
![alt text](./images/image-1.png)

### Task 3. OS 기본 설정

1. 모든 노드에서 OS 패키지를 업데이트하고 재시작합니다.

```bash
dnf update -y
reboot
```

2. 의존성 패키지를 설치합니다.

```bash
dnf install -y dnf-utils tar curl dnf-plugins-core lvm2 \
  device-mapper-persistent-data iscsi-initiator-utils device-mapper-multipath
```

3. 방화벽 서비스를 중지하고 비활성화합니다.

```bash
systemctl disable --now firewalld
```

4. Swap을 비활성화합니다.

```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
cat /etc/fstab
```

5. SELinux를 permissive로 변경합니다.

```bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
cat /etc/selinux/config
```
![alt text](./images/image-2.png)

6. 커널 모듈을 로딩하고 재부팅 후에도 적용되도록 설정합니다.

```bash
modprobe overlay
modprobe br_netfilter

cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

7. 커널 파라미터를 설정합니다.

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
EOF
```
```
sysctl --system
```

8. NTP 동기화를 위해 Chrony를 설치하고 상태를 확인합니다.

```bash
dnf install -y chrony
systemctl enable --now chronyd
```

```bash
chronyc sources -v
```
![alt text](./images/image-3.png)
```
timedatectl
```
![alt text](./images/image-4.png)

### Task 4. 컨테이너 런타임(Containerd) 설치

1. 모든 노드에서 Docker 저장소를 등록합니다.

```bash
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
dnf repolist --enabled
```

![alt text](./images/image-5.png)

2. Containerd를 설치하고 기본 설정 파일을 생성합니다.

```bash
dnf install -y containerd.io
```
```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

3. `SystemdCgroup` 값을 `true`로 변경합니다.

```bash
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep 'SystemdCgroup' /etc/containerd/config.toml
```

![alt text](./images/image-6.png)
4. Containerd 서비스를 시작하고 상태를 확인합니다.

```bash
systemctl enable --now containerd
systemctl status containerd
```
![alt text](./images/image-7.png)
### Task 5. 쿠버네티스 구성 요소 설치

1. 모든 노드에서 쿠버네티스 저장소를 등록합니다.

```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
```bash
dnf repolist --enabled
```
2. 저장소 등록을 확인 합니다.

![alt text](./images/image-8.png)

3. 쿠버네티스 구성 요소를 설치합니다.

```bash
dnf install -y kubelet kubeadm kubectl cri-tools --disableexcludes=kubernetes
```
```bash
kubectl version
kubeadm version
```
![alt text](./images/image-10.png)

4. Kubelet 서비스를 활성화합니다.

```bash
systemctl enable --now kubelet
systemctl status kubelet
```

> Note: 쿠버네티스 마스터가 구성되지 않아 현재 `kubelet` 서비스는 쿠버네티스 마스터와 통신할 수 없는 상태이므로 실패 상태일 수 있습니다. 다음 Task에서 쿠버네티스 마스터를 구성하면 정상 동작합니다.

### Task 6. 쿠버네티스 마스터 구성

1. 마스터 노드에서 `kubeadm` init yaml를 생성합니다.

```bash
kubeadm config print init-defaults > ~/kubeadm-init.yaml
vi ~/kubeadm-init.yaml
```

2. `kubeadm-init.yaml`에서 아래 항목을 수정합니다.

![alt text](./images/image.png)

| 항목 | 설명 |
| --- | --- |
| `localAPIEndpoint.advertiseAddress` | 마스터 노드 IP |
| `nodeRegistration.name` | 마스터 노드 호스트명 |
| `networking.podSubnet` | `10.224.0.0/16` |

> Note: `networking.podSubnet` 항목은 신규 추가 항목입니다.

3. 수정한 yaml 파일로 마스터 노드를 초기화합니다.

```bash
kubeadm init --config=kubeadm-init.yaml
```
![alt text](./images/image-11.png)

4. `kubectl` 사용 권한을 설정합니다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

5. 상태를 확인합니다.

```bash
kubectl version --output=yaml
kubectl get nodes
```

### Task 7. 워커 노드 조인

1. 마스터 노드에서 Join 명령어를 생성합니다.

```bash
kubeadm token create --print-join-command
```

> Note: 생성된 토큰은 24시간 후 초기화됩니다.

2. 워커 노드에서 생성된 Join 명령어를 실행합니다.

```bash
kubeadm join 192.168.102.129:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

3. 마스터 노드에서 노드 목록을 확인합니다.

```bash
kubectl get nodes
```

> Note: 워커 노드 조인을 하지 않은 시점에는 마스터 노드만 확인할 수 있습니다.

### Task 8. Calico CNI 배포

1. 마스터 노드에서 Calico를 배포합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/calico.yaml
kubectl get po -A
kubectl get nodes
```

> Note: CNI가 배포되기 전에는 노드 상태가 `NotReady`일 수 있습니다.
>
> Note: 진행 과정을 모니터링하려면 `watch -n 1 kubectl get po -A`를 실행합니다.

2. 모든 Pod가 `Running`, 모든 노드가 `Ready` 상태인지 확인합니다.

---

[처음으로](../../README.md) | [다음 LAB](../lab-02/kubernetes-addons.md)
