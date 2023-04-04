# API Version

`apiVersion` 

쿠버네티스는 모든 조작, 컴포넌트 간 통신, 외부 사용자의 명령을 

REST API 호출을 받아 API 서버에서 처리.

v1 : 대부분의 api가 포함되어 있는 첫 stable release API

apps/v1 : 디플로이먼트와 관련된 필드, replicas, strategy 등과 같은 필드를 가짐

각 kind에 따라 적합한 apiVersion 존재