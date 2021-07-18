# Rest Docs 도입 중 맞닥뜨린 asciiDoctor에 대한 에러

Spring Rest Docs를 하려고 gradle 설정을 하고 테스트도 통과시키고 ``src/docs/asciidoc/index.adoc`` 까지 준비를 마쳐놓은 상태였다. 그러고선 빌드를 하니깐 다음과 같은 오류가 나왔다.  

![image](https://user-images.githubusercontent.com/45073750/126065181-01b6fc4b-cf58-493f-bfe1-7dad7627fd31.png)

Type 'org.asciidoctor.gradle. xxx ' property 'xxx' is missing an input or output annotations.  

에러 내용에 어떤식으로 해결하라고 나와있는데 저걸 뭘 어떻게 해야될지도 모르겠고... 이전에 똑~~같이 Rest Docs 했는데 왜 안되나 싶었다. 그 이유는 버전 문제인 것 같았다.  

![image](https://user-images.githubusercontent.com/45073750/126065229-506e7c13-b2f9-4e45-b92f-b334e995cd57.png)

이곳에 존재하는 ``gradle-wrapper.properties`` 파일에 들어가서  

![image](https://user-images.githubusercontent.com/45073750/126065234-3c83abe1-099e-448f-a283-a7809d1784e0.png)

버전을 낮춰주었다. 그리고 이 버전을 낮추니깐 스프링 부트 버전도 낮추라는 문구가 나왔다. 내가 사용하고 있는 버전은 ``6.5.1``을 지원안해서 그런 것 같았다.  

![image](https://user-images.githubusercontent.com/45073750/126065263-13d703e7-22e7-4d4c-8e69-4c25f8b44a11.png)

이제 이렇게 해결되나 싶었더니, 또 다른 문제가 발생했다.  

![image](https://user-images.githubusercontent.com/45073750/126065272-e038b93a-7eac-4f67-85ce-1f21bfba5c4e.png)

``build/asciidoc/html5/index.html``은 생성이 되는데, 이 파일이 ``/resources/static/docs/`` 위치로 옮겨지지 않았다!  

```java
ext {
	snippetsDir = file('build/generated-snippets')
}

test {
	outputs.dir snippetsDir
	useJUnitPlatform()
}

asciidoctor {
	inputs.dir snippetsDir
	dependsOn test
}

bootJar {
	dependsOn asciidoctor
	from ("${asciidoctor.outputDir}/html5") {
		into "static/docs"
	}
}
```

gradle에 위와 같은 설정이 존재하는데, 테스트 후 스니펫들을 ``build/generated-snippets`` 위치로 보내고 그 후 asciidoctor 단계에서 이것들을 가지고 html을 만들어주고, bootJar 단계에서 ``${asciidoctor.outputDir}/html5``의 위치해 있는 파일을 ``static/docs``로 옮겨주어야 하는데 이 부분이 되고 있지 않은 것 같았다.  

bootJar 부분을 밑의 코드와 같이 수정했다.  

```java
task copyDocument(type: Copy) {
    dependsOn asciidoctor

    from file("build/asciidoc/html5/")
    into file("src/main/resources/static/docs")
}

build {
    dependsOn copyDocument
}
```

신기하게도 이제 ``resources/static/docs`` 위치에 index.html 이 들어오게 된다!! build 대신 bootJar를 입력해도 수행이 되는 것을 확인할 수가 있었다.  

아마 gradle 버전에 따른 지원 차이인 것 같다고 추측 중이다. 🤔  

***