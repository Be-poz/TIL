# k8s secret properties 파일로 volume mount 하기

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bepoz-secret
  labels:
    app: bepoz-app
type: Opaque
data:
  name: YmVwb3o= # 'bepoz'의 base64 인코딩
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bepoz-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bepoz-app
  template:
    metadata:
      labels:
        app: bepoz-app
    spec:
      containers:
      - name: busybox
        image: busybox:latest
        command: ["sleep", "200"]
        volumeMounts:
        - name: secret-volume
          mountPath: "/etc/secret-volume"
      volumes:
      - name: secret-volume
        secret:
          secretName: bepoz-secret
```

일반적으로 secret을 volume 등록하고 mount 할 때 위와 같이 한다.  

이렇게 등록하게되면 mountPath에 아래와 같이 key가 파일이름, 그리고 내부 내용이  value로 들어가게 된다.  

```sh
/ # ls
bin    dev    etc    home   lib    lib64  proc   root   sys    tmp    usr    var
/ # cd /etc/secret-volume
/etc/secret-volume # ls
name
/etc/secret-volume # cat name
bepoz
```

그런데 이 secret 값을 properties 형식으로 저장을 해야만 하는 상황이 있었다.  
컨테이너의 COMMAND 명령어로 secret 들을 읽어서 properties 파일로 만들게끔도 해봤는데, 파드 내부에서 진행했을 때에는 되었는데 이상하게도 계속 오류가 발생했다. 그래서 아예 secret을 mount 할 때 properties로 들어가게끔 하는 방법을 택했다.  

방법은 사실 간단하다. key 값이 파일명이 되므로 이 key 값에 파일명을 입력하고 value에는 properties 내용을 작성하는 것이다.  

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bepoz-secret
  labels:
    app: bepoz-app
type: Opaque
data:
  info.properties: bmFtZTprYW5nCmFnZToxMDA= 
```

위의 value 값은  

```
name:bepoz
age:100
```

이 값을 그대로 인코딩한 것이다. 여기서 value에 따옴표가 들어가면 문제가 생길 수 있다.  

```yaml
/etc/secret-volume # cat info.properties
name:kang
age:100
```

pod에 들어가서 mountPath에서 해당 secret을 확인해보면 적절하게 properties 파일로 되어있는 것을 확인할 수 있다.  

---

