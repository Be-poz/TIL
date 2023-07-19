# 여러 파드 로그 조회 명령어 Stern

``k logs {pod-name}`` 으로 파드를 조회할 때에 1개의 파드만 조회가능하다. 만약 replicas가 여러개라면 1개의 파드를 보는 것으로는 제대로된 로그파악이 되지 않는다.  

``k logs -l app=bepoz`` 이렇게 레이블이 ``app=bepoz``인 것 들을 한 번에 조회할 수도 있긴하지만 ``-f`` 옵션으로 지속적으로 로그변경을 확인하고 싶을 때에는 최대 5개의 파드까지만 가능하기 때문에 파드 수가 많다면 ``k logs -f -l app-bepoz`` 이렇게 사용이 어렵다.  

명령어 stern은 가능하다. stern은 쿠버네티스 클러스터의 여러 파드와 여러 컨테이너를 ``tail`` 할 수 있게끔 해준다.  

``brew install stern`` 으로 stern을 설치한다.  

기본적으로 ``stern pod-query [flags]``의 형태로 명령어를 사용한다.  

``stern bepoz -n bepoz-namespace --exclude-container istio`` 왼쪽과 같이 입력하면 bepoz-namespace 네임스페이스에서 bepoz 이름이 들어간 파드들의 로그를 확인하는데 istio 이름이 들어간 컨테이너는 배제시키는 명령어다.

여러 옵션과 사용법이 있기 때문에 자세한 사용법은 https://github.com/stern/stern 공식 README를 참조하자.  

---

