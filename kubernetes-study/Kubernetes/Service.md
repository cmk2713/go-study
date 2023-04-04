# Service

# 가상 IP와 서비스 Proxy

> 쿠버네티스 클러스터의 모든 노드는 **kube-proxy**를 실행하며 **kube-proxy**는 ExternalName 이외의 유형의 서비스에 대한 가상 IP 형식을 구현
> 

# Kube Proxy

> Go 언어로 작성된 애플리케이션으로써, 클라이언트의 요청을 알맞은 백엔드 Pod로 라우팅함
> 

## 동작 과정

Kube-proxy가 netfilter를 이용해 서비스의 IP로 들어오는 패킷을 kube-proxy 자신에게 라우팅되도록 설정함.

kube-proxy로 들어온 요청을 실제 server Pod의 IP:Port로 전달(iptable이 DNAT(Destination NAT) 를 이용해 전달)

## Kube Proxy의 모드 3가지

- **User Space 프록시 모드**
    - 서비스의 Cluster IP와 포트에 대한 트래픽을 캡처하는 iptables 규칙을 설치해 kube-proxy가 수신하고 있는 포트로 해당 트래픽을 리다이렉션 후 요청을 받으면 알맞은 백엔드 Pod를 선택해 전달
    - User Space에서 동작하기 때문에 Kernel Space와 User Space 사이에서 패키지를 복사하는 과정이 필요 ⇒ Proxy Process에 추가적인 지연시간 존재
    - 기본적으로, 라운드 로빈 알고리즘으로 백엔드 Pod를 선택
- **iptables 프록시 모드**
    - 위의 kernel ↔ user 간의 복사과정을 피하기 위해 kube-proxy는 iptables 모드에서 작동
    - Cluster IP로 요청이 전달되면 iptables는 DNAT을 이용해 알맞은 백엔드 Pod로 직접 전달
- **ipvs(ip virtual server) 프록시 모드**
    - Hash Table을 이용해 규칙을 저장 ⇒ 수 천개 이상의 서비스가 있는 대규모 클러스터에서는 ipvs 모드가 성능이 더 우수
    - 기존의 Load balancing 알고리즘보다 더 다양한 알고리즘 제공

# Cluster IP

> 클러스터 내부에서만 사용
> 
- 클러스터 안 노드나 파드가 Cluster IP를 이용해 서비스에 연결된 각 파드들에 접근
- 외부에서는 접근 불가

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cluster-ip
  namespace: namespace
spec:
  type: ClusterIP
  selector:
    app: #pod label:app
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
      name: http
```

<aside>
💡 ⇒ Pod는 클러스터의 노드 간에 동적으로 생성, 삭제 및 마이그레이션 
⇒ Pod는 일시적이며 종료 후 재생성 시 IP 변경 (계속 변경될 우려 발생)
⇒ ClusterIP로 Pod를 바인딩하여 문제 해결

</aside>

# NodePort

> Port 번호를 통해 외부에서 접근
> 
- 모든 Node에 특정 포트를 열어 두고, 이 포트로 보내지는 모든 트래픽을 서비스로 포워딩함
- nodeport의 범위는 `--service-node-port-range` 플래그로 지정된 범위(기본값 : 30000 ~ 32767)
- NodePort로 전송되는 트래픽을 `Kube-proxy` 가 가져와 연결된 포드들로 redirect
    
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: test-nodeport
      namespace : namespace
    spec:
      type: NodePort
      selector:
        app: #포드 app:name
      ports:
        - port: 8080
          targetPort: 8080
          nodePort: 32540 #대부분의 경우 Kubernetes가 포트를 선택하도록 두어야 함
    ```
    

---

[NodePort 과정](Service%201f83ad51de0a4ddb956792a766d53cee/NodePort%20%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A5%E1%86%BC%2089015682669e4ebc9427baed6eafb43d.md)

### Production 에 부적합

- 외부 사용자가 표준 포트번호로 접속 불가
- 각 서비스당 하나의 포트 사용 ⇒ 서비스 수가 많을수록 포트가 부족

# Load Balancer

> 외부의 로드밸런서를 사용
> 
- 서비스에 대한 로드 밸런서를 프로비저닝
- 외부에서 파드 접속 가능
- 로드 밸런서 유형의 서비스는 로드 밸런서를 생성하기 위한 요청일 뿐이며, 실제 작업은 클라우드 공급자가 수행

- LoadBalancer로 노출하고자 하는 각 서비스마다 자체의 IP주소를 갖게됨
- 노출하는 서비스마다 LoadBalancer 비용을 지불

# External Name

> 클러스터 내부에서 외부로 접근할 때 사용
> 
- 서비스를 `.spec.externalName` 필드에 설정한 값과 연결

# Ingress

> 서비스 X. 여러 서비스들 앞에서 스마트 라우터 역할을 하거나 클러스터의 진입점 역할
> 
- Ingress Controller는 Cluster 내부에 Pod로 배포되기 때문에 외부에서 직접 Access할 수 없음
    
    ⇒NodePort와 LoadBalancer와 함께 작동해 외부 트래픽이 클러스터로 들어가는 전체 경로를
    제공해야 함
    
- 여러 능력을 가진 Ingress 컨트롤러 타입이 있음
    - 기본 GKE Ingress 컨트롤러 : HTTP(S) Load Balancer를 만들어줌
        
        ⇒ 경로와 서브 도메인 기반의 라우팅 모두 지원( sub.xxx.com, [xxx.com/path/](http://xxx.com/path/) )
        

# Istio

> 
>