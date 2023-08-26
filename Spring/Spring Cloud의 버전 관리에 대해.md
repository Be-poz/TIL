# Spring Cloud의 버전 관리에 대해

Spring Cloud 버전의 선택은 현재 사용하고 있는 Spring Boot의 버전을 따라간다.  

<img width="909" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/fb577ab9-1c65-4f50-a965-915ea4d28086">

https://spring.io/projects/spring-cloud  

공식문서를 살펴보면 위와 같이 나와있다. 현재 내가 사용하고 있는 boot의 버전이 3점대라면 ``2022.0.x``를 사용하면 될 것이다.  

Spring Cloud 프로젝트에는 세부적으로 16개의 프로젝트가 있고 각기 다른 버전을 가지고 있기 때문에 버전간 원활한 호환을 위해서 Spring Cloud 팀이 위의 사진처럼 release train을 따로 두어 버전을 각각 관리하는 것이 아니라 한 번에 관리할 수 있게끔 해준다.  

```gradle
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudDependenciesVersion}"
    }
}

dependencies {
    implementation 'org.springframework.cloud:spring-cloud-stream-binder-kafka'
}
```

위와 같이 ``dependencyManagement``에 버전을 정해두면 ``dependencies`` 에서 사용되는 Spring Cloud의 프로젝트들의 의존성이 대표 버전을 따라가게 된다.  

```gradle
buildscript {
    ext {
        springCloudDependenciesVersion = "2021.0.3"
    }
}
```

위와 같이 부모 gradle에서(멀티 모듈 가정 하) 21.0.3 버전으로 설정을 해놨으면,  

<img width="395" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/b16c7c4e-1537-45ed-9a4a-21e2b2f71cb5">

https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-2021.0-Release-Notes#202103

위 링크에 나와있는 대로 버전이 들어가게 된다. 만약 해당 월에 내가 사용하고 있는 의존성에 대한 버전이 나와있지 않다면 가장 최신의 과거에 적혀져있는 버전으로 들어가게 된다.  

위의 버전 상태로  ``'implementation 'org.springframework.cloud:spring-cloud-stream-binder-kafka'``다음 과 같이 cloud-stream 의존성을 추가하게되면 

<img width="466" alt="image" src="https://github.com/Be-poz/TIL/assets/45073750/a050032d-e9fb-4246-95ab-67291f33d9f4">

알맞게 ``3.2.4`` 버전이 추가된다.  

---

### REFERENCE

https://spring.io/projects/spring-cloud  

https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-2021.0-Release-Notes#202103

