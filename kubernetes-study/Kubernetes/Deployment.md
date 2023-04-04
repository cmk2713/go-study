# Deployment

# 디플로이먼트(Deployment)

> 파드와 레플리카셋에 대한 선언적 업데이트 제공
> 

<aside>
💡 **의도하는 상태**를 설명하고, 컨트롤러를 통해 현재 상태에서 의도하는 상태로 비율을 조정하며 변경

</aside>

---

- 새 레플리카셋을 생성하는 디플로이먼트를 정의 or 기존 디플로이먼트를 제거하고, 모든 리소스를 새 디플로이먼트에 적용

### .yaml

```yaml
apiVersion: apps/v1 #오브젝트를 생성하기 위해 사용하고 있는 쿠버네티스 API 버전
kind: Deployment #오브젝트 종류

metadata: #오브젝트를 유일하게 구분지어줄 데이터
 name: DeploymentName #Deployment 이름
 namespace: namespace  #네임스페이스
 labels: #레이블 설정
  app: app-name
spec: #오브젝트에 어떤 상태를 의도하는지
 progressDeadlineSeconds: 600 #진행이 정지되었음을 나타내는 디플로이먼트 컨트롤러가 대기하는 시간 #데드라인 넘을 시 .status.conditions 속성에 컨디션 추가
 replicas: 2  #pod 생성 수
 revisionHistoryLimit: 10 # 디플로이먼트가 유지해야 하는 이전 replica 수(for 롤백), 10이 디폴트
 selector:  #Deployment의 대상이 되는 파드를 찾는 필수 필드, 앱컨테이너의 레이블을 식별하여 해당되는 파드 관리
						#.spec.selector 와 .spec.template.metadata.labels가 일치해야함
  matchLabels: #{"key":"value"}
   app: app-name
 strategy: #배포전략
    type: RollingUpdate #
    rollingUpdate: # 롤링업데이트,
      maxSurge: 25% #의도한 파드 수에 대해 생성할 수 있는 최대 파드 수, maxUnavailable이 0이면 0 불가능, 25%가 디폴트
      maxUnavailable: 25% #업데이트 프로세스 중 사용할 수 없는 최대 파드 수, maxSurge가 0이면 0 불가능, 25%가 디폴트
 template:
  metadata:
   labels: #파드에 레이블 붙임
    app: app-name
  spec: #파드 템플릿 사양 or 파드가 실행하는 이미지
   containers:
   - name: app-name
     image: docker-image #이미지 
						#imagePullPolicy 필드 (pull 정책 : IfNotPresent(디폴트), Always, Never)
						#imagePullPolicy 필드 생략 & 이미지 태그= :latest  => Always
						#imagePullPolicy 필드 생략 & 이미지 태그= 명시x  => Always
						#imagePullPolicy 필드 생략 & 이미지 태그 != :latest  => IfNotPresent
     ports:
     - containerPort: 8080 #For documentation
     envFrom:  #환경변수
     - secretRef: #Secret 사용
         name: secret
     volumeMounts: # 볼륨
     - name: config
       mountPath: "/mountpath"
       readOnly: true #볼륨을 읽기 전용으로 ControllerPublished(연결)할지 여부
											# default = false
     livenessProbe: #컨테이너 체크 : Probe => livenessProbe : 컨테이너의 동작 여부
       failureThreshold: 3 #실패가 처음 기록된 후 이 횟수만큼 프로브 재시작, 3이 디폴트
       initialDelaySeconds: 0 #컨테이너가 시작되는 시간과 프로브가 처음 실행되는 시간 사이의 지연
       httpGet: #컨테이너 IP주소에 대한 HTTP GET 요청 수행 (200 <= status <=400 : 성공)
         path: /v1/health #지정 경로
         port: 8080 #지정 포트
         scheme: HTTP 
       periodSeconds: 10 #초기 지연 후 프로브가 실행되는 빈도, 10이 디폴트
       successThreshold: 1 #비정상 컨테이너를 정상 상태로 되돌리기 위한 기준. (연속 활성 검사 수), 1이 디폴트
       timeoutSeconds: 1 #각 프로브는 이 시간이 지나면 시간이 초과되고 실패한 것으로 표시, 1이 디폴트
   imagePullSecrets: #secret
   - name: icr  
   volumes:
   - name: config
     configMap: #일반 config
       name: icr-manager-configmap
       items:
       - key: "config.yaml"
         path: "config.yaml"
```

<aside>
💡 `spec.containers.ports.containerPort` 명시하는 이유? ⇒ 문서화

</aside>

## Pod

> 쿠버네티스에서 생성하고 관리할 수 있는 배포 가능한 가장 작은 컴퓨팅 단위
> 
- docker image를 실행시킨 컨테이너를 감싸고 있는 단위
- 한 파드 내에 여러 개의 컨테이너 존재 가능
- 컨테이너 간에 localhost로 통신 가능, but 같은 포트 사용 불가

## Replica set

- 필드에 지정된 설정을 충족하기 위해 필요한 만큼 파드를 만들고 삭제
- 지정된 수의 파드 replica가 항상 실행되도록 보장
- 업데이트 기능 제공 x

⇒ 사용자 지정 오케스트레이션이 필요하거나 업데이트가 전혀 필요하지 않은 경우라면 레플리카셋을 직접 사용하는 것보다 디플로이먼트를 사용하는 것을 권장함

## Deployment

- 디플로이먼트 생성 시 Replica Set도 함께 생성
- 디플로이먼트는 레플리카셋의 상위 개념으로 레플리카셋을 관리하고 다른 유용한 기능과 함께 파드에 대한 선언적 업데이트를 제공

### 기능

- 디플로이먼트 yaml에 변경이 생겨 레플리카셋의 파드들을 전부 변경 시 디플로이먼트의 기록으로 남음 ⇒ 이전 버전으로 롤백이 가능
- 업데이트 전략
    - 롤링업데이트 : 무중단 배포, 새 버전 파드를 하나씩 늘려가고 기존 버전의 파드를 하나씩 줄여나가는 방식
    - Recreate : 기존 파드를 모두 삭제 후 새 파드 생성, 다운타임 발생
    - Blue/Green : 신, 구버전 2가지 모두 서버에 마련 후 한번에 교체
    - Canary : 구버전과 신버전을 구성해 일부 트래픽을 신버전으로 분산시켜 테스트 진행 후 서서히 옮기는 방식 ⇒ 위험감지

## rollback

`kubectl rollout undo deployment/nginx-deployment`

## scaling

`kubectl scale deployment/nginx-deployment --replicas=10`

# Volumes

<aside>
💡 **컨테이너에 문제가 발생해 컨테이너가 삭제된다면 데이터도 또한 같이 삭제된다. 
로그 파일이나 데이터 베이스와 같은 경우 사라지면 큰 장애가 발생하므로
이러한 경우에 Volume을 사용해 데이터를 보관해 주어야 함**

</aside>

---