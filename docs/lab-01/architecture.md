# 교육환경 구성도

Portworx Technical Workshop 교육환경의 기본 구성입니다.

```mermaid
flowchart TB
  user["교육생 / 브라우저"]
  vip["MetalLB VIP<br/>192.168.102.xxx"]

  subgraph k8s["Kubernetes Cluster"]
    direction TB

    master["Master Node<br/>px-lab-xx-m01<br/>Control Plane"]

    subgraph workers["Worker Nodes"]
      direction LR
      w1["Worker Node 1<br/>px-lab-xx-w01"]
      w2["Worker Node 2<br/>px-lab-xx-w02"]
      w3["Worker Node 3<br/>px-lab-xx-w03"]
    end

    subgraph ngf["NGINX Gateway Fabric"]
      gateway["Gateway<br/>nginx-gateway"]
      route["HTTPRoute<br/>px-bbq.xx.px.lab"]
    end

    subgraph app["Application Namespace"]
      svc["Service<br/>px-bbq"]
      pod1["px-bbq Pod"]
      pod2["px-bbq Pod"]
      pvc["PVC<br/>Portworx-backed"]
    end

    subgraph px["Portworx Storage Layer"]
      sc["StorageClass<br/>px-postgres / Portworx"]
      pv["PV / Volume"]
      pxcluster["Portworx Cluster<br/>3 Worker Nodes"]
    end
  end

  user -->|"HTTPS / HTTP"| vip
  vip -->|"LoadBalancer IP"| gateway
  gateway --> route
  route --> svc
  svc --> pod1
  svc --> pod2
  pod1 -->|"mount"| pvc
  pod2 -->|"mount"| pvc
  pvc --> pv
  sc -. "provisions" .-> pv
  pv --> pxcluster

  master -. "kube-apiserver / control" .-> workers
  pxcluster -. "replicated storage" .-> w1
  pxcluster -. "replicated storage" .-> w2
  pxcluster -. "replicated storage" .-> w3
```

## 구성 요약

- Control Plane: 마스터 노드 1대
- Worker: 워커 노드 3대
- External Access: MetalLB VIP가 `LoadBalancer` 진입점 제공
- Ingress/Gateway: NGINX Gateway Fabric이 `px-bbq` 앱 라우팅
- Application: `px-bbq` 서비스와 Pod가 백엔드 PVC를 마운트
- Storage: Portworx가 PVC/PV를 프로비저닝하고 워커 노드 기반 스토리지 클러스터로 데이터 제공

---

[처음으로](../../README.md) | [Lab 01](./kubernetes-cluster.md)

