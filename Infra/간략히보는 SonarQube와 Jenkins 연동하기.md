# 간략히보는 SonarQube와 Jenkins 연동하기

1. EC2를 하나 파서 도커를 설치한다.  

   ```shell
   sudo apt-get update && \
   sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common && \
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - && \
   sudo apt-key fingerprint 0EBFCD88 && \
   sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" && \
   sudo apt-get update && \
   sudo apt-get install -y docker-ce && \
   sudo usermod -aG docker ubuntu && \
   sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && \
   sudo chmod +x /usr/local/bin/docker-compose && \
   sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
   ```

2. ``sonarqube``를 도커위에 올린다.  

   ```shell
   docker run -d --name sonarqube -p 8000:9000 sonarqube
   //밑의 스샷은 9000:9000 인데 실제로는 8000:9000 으로 실행했음
   ```

   <img width="837" alt="스크린샷 2021-08-06 오후 5 18 26" src="https://user-images.githubusercontent.com/45073750/128480205-025a35a1-99d6-407b-8428-71db41d3a860.png">

   <img width="931" alt="스크린샷 2021-08-06 오후 5 18 40" src="https://user-images.githubusercontent.com/45073750/128480197-686dacc1-0ce2-4621-aaec-73df39129ee7.png">

3. ``http://{sonarqube_ip}:8000/sonar`` 에 접속한다. 초기 id/pw는 admin/admin 이다.  

4. 토큰을 생성하고, 프로젝트를 만든다.  

   ![image](https://user-images.githubusercontent.com/45073750/128464342-e5040c61-1156-4304-8aa6-33dd19a1cd16.png)

   ![image](https://user-images.githubusercontent.com/45073750/128464379-96ae9dba-190c-4ea3-9373-49b0a391e990.png)

   ![image](https://user-images.githubusercontent.com/45073750/128464581-65d37b29-2700-4202-b116-ed1a41d5e4df.png)  

5. 젠킨스에서 SonarQube 플러그인을 설치한다.  

![image](https://user-images.githubusercontent.com/45073750/128463219-68ed28f9-7ff7-4f87-ada4-2871638987fc.png)

6. 젠킨스 메인화면 > Jenkins 관리 > Global Tool Configuration에서 ``SonarQube Scanner`` 를 설정해준다.  

   ![image](https://user-images.githubusercontent.com/45073750/128478246-95a2f653-bf3f-4f40-bd06-02229b80f4d1.png)

7. 젠킨스 메인화면 > Jenkins 관리 > 시스템 설정에서 ``SonarQube servers`` 에 소나큐브 url을 등록해준다.  

   ![image](https://user-images.githubusercontent.com/45073750/128478455-91b8dedb-b825-4804-8c3e-55bbe4620049.png)

8. 자바 프로젝트에 Jacoco 설정을 해준다.  

   ![image](https://user-images.githubusercontent.com/45073750/130346131-d19d58ab-d7e0-4439-b9d7-72b4b963e3e5.png)

   ```gradle
   jacocoTestReport {
       dependsOn bootJar
   
       reports {
           html.enabled true
           xml.enabled true
           csv.enabled false
   
           html.destination file("src/jacoco/jacoco.html")
           xml.destination file("src/jacoco/jacoco.xml")
       }
   
       afterEvaluate {
           classDirectories.setFrom(files(classDirectories.files.collect {
               fileTree(dir: it,
                       exclude: [
                               '**/TyfServerApplication*'
                       ])
           }))
       }
   
       finalizedBy 'jacocoTestCoverageVerification'
   }
   
   jacocoTestCoverageVerification {
       violationRules {
           rule {
               enabled = true
               element = 'CLASS'
   
               limit {
                   counter = 'METHOD'
                   value = 'COVEREDRATIO'
                   minimum = 0.6
               }
   
               excludes = [
                       '**.*TyfServerApplication*'
               ]
           }
       }
   ```
   
   ``jacocoTestReport`` 에서 ``excludes`` 는 분석대상에서 제외하는 것,  
   ``jacocoTestCoverageVerification`` 에서 ``excludes`` 는 테스트 대상에서 제외  
   
9. 젠킨스 item 구성의 Build 탭에서 ``Execute SonarQube Scanner`` 를 등록해준다.  

   ![image](https://user-images.githubusercontent.com/45073750/130346164-fd84d89b-0792-4370-b81e-d9f9ab578863.png)

   ![image](https://user-images.githubusercontent.com/45073750/128463939-50c2ceac-85bd-4880-8e95-bed80c53854d.png)

   ``Analysis properties`` 에는 다음과 같이 작성  

   ```
   sonar.login = {소나큐브에서 발급한 token}
   sonar.projectKey = {소나큐브에서 설정한 key 이름}
   sonar.projectName= {소나큐브에서 설정한 project 이름}
   sonar.host.url = {소나큐브 url}
   sonar.report.export.path = sonar-report.json
   sonar.sources=server/src/main/java, server/src/main/resources
   sonar.java.coveragePlugin = jacoco
   sonar.coverage.jacoco.xmlReportPaths= server/src/jacoco/jacoco.xml (자코코 xml 생성 위치)
   sonar.test.inclusions=**/*Test.java (include 할 클래스들)
   sonar.exclusions=**/*TyfServerApplication.java, */Q*.java (exclude 할 클래스들)
   sonar.projectVersion=1.0
   sonar.sourceEncoding=UTF-8
   sonar.java.binaries=server/build/classes
   ```

   각 속성 값들에 대한 정보는 https://sonarqubekr.atlassian.net/wiki/spaces/SON/pages/388027 이곳에서 확인가능    

***