# Secret, ConfigMap

# Secret

<aside>
💡 **보안이 중요한 패스워드나 API키, 인증서 파일 등 저장**

</aside>

- key : value 에서 value를 base64로 인코딩 ⇒ **바이너리 파일 저장 지원**
- 항상 메모리에 저장되어 있기 때문에 상대적으로 접근 어려움
- 하나의 secret 사이즈는 최대 1M까지 지원, But **메모리에 지원되는 특성 때문에 여러 개 저장 시 API Server 나 노드에서 이를 저장하는 Kubelet의 메모리 사용량이 늘어나 Out Of Memory 이슈 유발** ⇒ 꼭 필요한 정보만 저장

# ConfigMap

<aside>
💡 **일반적인 환경 설정 정보, CONFIG 정보 등 저장**

</aside>

- Secret과 기본적으로 거의 유사
- key : value 값을 일반 문자열로 저장

### 참조

> [https://bcho.tistory.com/1268](https://bcho.tistory.com/1268)
>