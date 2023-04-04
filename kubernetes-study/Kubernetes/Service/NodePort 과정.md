# NodePort 과정

![Untitled](NodePort%20%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A5%E1%86%BC/Untitled.png)

**{공인IP}:31051** 로 접속 시 

![Untitled](NodePort%20%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A5%E1%86%BC/Untitled%201.png)

**Cluster IP**인 10.97.205.121이 받아

![Untitled](NodePort%20%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A5%E1%86%BC/Untitled%202.png)

**Pod 속 EndPoint** 인 10.1.0.10 , 10.1.0.11 로 전송( 파드 2개 ) 

![Untitled](NodePort%20%E1%84%80%E1%85%AA%E1%84%8C%E1%85%A5%E1%86%BC/Untitled%203.png)