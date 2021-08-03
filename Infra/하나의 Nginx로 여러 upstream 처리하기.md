# 하나의 Nginx로 여러 upstream 처리하기

```groovy
events {}

http {
  upstream app {
    server 앱_아이피;
  }

  upstream api-dev {
    server 데브서버_아이피;
  }

  upstream api {
    server 프로드서버_아이피;
  }

  # Redirect all traffic to HTTPS

//  server {
//    listen 80;
//    return 301 https://$host$request_uri;
//  }
//
//  server {
//    listen 443 ssl;

//    location / {
//      proxy_pass http://app;
//    }
//  }

 server {
    listen 80;
    server_name api-dev.thankyou-for.com;
    return 301 https://$host$request_uri;
  }

 server {
    listen 443 ssl;
    server_name api-dev.thankyou-for.com;

    location / {
      proxy_pass http://api-dev;
    }

  }

  server{
    listen 80;
    server_name api.thankyou-for.com;
    return 301 https://$host$request_uri;
  }
 server {
    listen 443 ssl;
    server_name api.thankyou-for.com;

    location / {
      proxy_pass http://api;
    }

  }
}
```

TLS에 대한 설정들은 모두 뺐다. 하나의 Nginx로 여러 upstream으로 보내주기 위한 설정파일이다.  

밑에서 부터 살펴보면 80포트로 ``api.thankyou-for.com``으로 들어오면 그대로 https로 변환하여서 리다이렉 해준다. 그러면 바로 밑에서 받게된다. ``listen 443 ssl; server_name api.thankyou-for.com`` 여기에서 말이다.  

그 위의 ``server_name``이 ``api-dev.thankyou-for.com``에서도 마찬가지다.  

그렇다면 server_name이 그 어떤 것에도 포함되지 않는다면 어떤 것으로 처리가 될까??  
현재 상황에 대한 조건은 다음과 같다. 특정 도메인을 구매했고, ``api.thankyou-for.com``과 ``api-dev.thankyou-for.com``은 해당 Nginx가 존재하는 ec2의 public_ip를 가리키고 있다. 이를 받기위해서 위에 server_name을 두어서 받고 있는 것이다. 그런데 내가 해당 도메인 싸이트에서 위 2개의 url이 아닌 ``bepoz.thankyou-for.com``도 해당 Nginx를 받게끔 했다고 한다면 어떻게 되는지가 궁금한 것이다.  

결론부터 말하자면 매칭되는 ``server_name``이 없을 경우에는 가장 위쪽에 있는 서버를 ``defualt_server``로 보고 연결해준다. 즉, 위의 설정 파일에서 ``api-dev.thankyou-for.com``에서 처리하게 된다. 주석을 풀게된다면 해당 서버가 받게 될 것이다.  

***

### REFERENCE

http://nginx.org/en/docs/http/request_processing.html