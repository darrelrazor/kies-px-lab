# Lab 05. Portworx 기본 운영

### Task 1. Portworx 클러스터 정보 조회

1. `pxctl1 cluster list` 명령어로 Portworx 클러스터 목록을 확인합니다.

```bash
pxctl1 cluster list
```

2. `pxctl1 cluster inspect <Node_ID>` 명령어로 Portworx 클러스터를 구성하는 첫 번째 워커 노드의 상세 정보를 확인합니다.

```bash
pxctl1 cluster inspect <Node_ID>
```

3. `pxctl1 cluster provision-status` 명령어로 클러스터의 Provision 상태를 확인합니다.

```bash
pxctl1 cluster provision-status
```

4. 클러스터 개요를 확인합니다.

```bash
pxctl1 status
```

### Task 2. Volume 기본 관리

1. `pxctl1 volume create` 명령어로 볼륨을 생성합니다.

```bash
pxctl1 volume create vol1
```

2. `pxctl1 volume list` 명령어로 볼륨 목록을 확인합니다.

```bash
pxctl1 volume list
```

3. `pxctl1 volume inspect` 명령어로 해당 볼륨의 상세 정보를 확인합니다.

```bash
pxctl1 volume inspect vol1
```

---

[처음으로](../../README.md) | [이전 LAB](../lab-04/portworx-deployment.md) | [다음 LAB](../lab-06/dynamic-volume-provisioning.md)
