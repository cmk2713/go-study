# imagePullSecrets

<aside>
💡 Private 한 이미지를 받아올 때, 컨테이너 레지스트리에 인증하기 위해 사용

</aside>

> ***Github Container Registry***
> 
1. Github의 **Personal Access Token** 생성
2. `echo -n "username:Token" | base64` ⇒ 3
3. `echo -n  '{"auths":{"ghcr.io":{"auth":"2의 결과"}}}' | base64` ⇒ 4
4. yaml 생성
    
    ```yaml
    apiVersion: v1
    data:
      .dockerconfigjson: #3의 결과
    kind: Secret
    metadata:
      name: #secret명
      namespace: #네임스페이스
    type: kubernetes.io/dockerconfigjson
    ```
    
5. Deployment 소스 추가
    
    ```yaml
    spec:
    	imagePullSecrets:
         - name: #secret명
    ```