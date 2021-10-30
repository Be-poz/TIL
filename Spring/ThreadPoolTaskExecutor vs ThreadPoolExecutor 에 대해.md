# ThreadPoolTaskExecutor vs ThreadPoolExecutor 에 대해

 ``Executors``를 사용해서 스레드 관리를 해줄 수가 있었다. [링크](https://github.com/Be-poz/TIL/blob/master/Java/ExecutorService%20%EC%99%80%20ThreadPoolExecutor%20%EC%97%90%20%EB%8C%80%ED%95%B4.md)  
해당 클래스에서 사용하는 ``Executors.newFixedThreadPool()`` 는 내부적으로 ``ThreadPoolExecutor`` 를 사용하고 있다.  
그래서 문득 ``ThreadPoolExecutor`` 와 ``ThreadPoolTaskExecutor`` 의 차이점이 궁금해졌다.  

  > Spring을 사용한다면 ThreadPoolTaskExecutor를 살펴보는 것도 좋다. 내부 구현은 ThreadPoolExecutor로 구현되어 있다. ThreadPoolExecutor 보다 조금 더 간편하며, 추가적인 return 타입도 있다. 

  출처 http://wonwoo.ml/index.php/post/2254  
  spring에서 조금 더 편하게 만든 스레드 관리 클래스였다.  
![image](https://user-images.githubusercontent.com/45073750/138734176-4312fc6b-7870-4120-958e-0de445c711fc.png)
  내부적으로 ``ThreadPoolExecutor``를 사용하고 있었다.  

---

### REFERENCE

http://wonwoo.ml/index.php/post/2254