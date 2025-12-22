---
title: "[티켓 예매 서비스] 테스트서버 배포"
excerpt: " "

categories:
  - large-traffic-service
tags:
  - [tag1, tag2]

permalink: /categories/projects/large-traffic-service/11

toc: true
toc_sticky: true

date: 2025-12-18
last_modified_at: 2025-12-19
---  
  
## 배포  
  
이제 나의 작고 소듕한 서버를 배포를 할 것이다.  
앞서 말한 대로 AWS를 사용하여 최대한 저렴하게 프로젝트를 진행할 생각이다.  
프론트코드는 짜다가 생각보다 백 쪽에서 잡아먹는 시간이 많아 선택과 집중으로 백에 좀 더 집중하기로 했다.  
  
### AWS  
  
<img src="/assets/images/posts/largeTrafficService/ec2setting.png" width="100%" height="50%">  
  
t3.micro에 EBS는 20GiB이다.  
설명에 따르면 t3.micro는 **버스트 가능** 인스턴스로 부하가 많을 때 순간적으로 크래딧을 소모하여 성능을 끌어올린다고 한다.  
하지만 그 이상 넘어가면 자동적으로 시스템을 멈춘다고 하니 티켓판매 사이트와 같이 부하가 많이 걸리는 사이트를 서비스하면 대참사가 날 확률이 높다.  
  
나는 테스트환경이고 제한된 환경에서 서버의 성능을 올리는 방법을 찾는 것이기때문에  
애매하게 살아남는 것이 아니라 확실하게 죽어주면 오히려 좋다.  
어느정도의 부하를 걸면 견뎌내는지에 대한 기준점을 찾기 좋기 때문이다.
  
다만...
Redis를 적극적으로 사용하려고 했는데 t3.micro에서 지원하는 램이 1GiB인게 조금 걸리지만 일단 시도해본다...    

인스턴스를 생성하고 접속하면 오랜만에 보는 반가운 창이 뜬다.  
  
노드를 다뤘을 때에는 nvm과 pm2 등등 서비스에 필요한 구성 요소들을 다운받는 스크립트를 짜놨었는데  
(어디 갔는지 모르겠다. 이전 회사 개발서버 깊숙이 어딘가에 있을 것 같다...)  
자바는 jar파일로 업로드하고 내장 톰캣으로 돌리니 자바만 설치하고 자동배포만 걸어주면 알아서 돌아가는 게 신기하다.  
  
우선 ~/app/ticketpark 를 만들어 빌드된 서버파일이 있을 디렉토리를 만들고  
WS와 로드밸런싱을 위한 NginX를 설치하자.  
    
---
  
### NGINX  
  
  
``` bash
server {
    listen 80;
    server_name _; 

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```
  
NginX는 이전에 실무에서 몇 번 세팅을 해본 경험이 있다.  
Apache와는 다르게 훨씬 보기 편하고(개인적으로) 세팅도 간편한 느낌이다.  
  
현재는 운영서버가 곧 개발서버이고, 프론트는 개발 중에 있으니 location은 하나만 잡아주면 된다.  
  
이제 중요한 workflow를 설정을 해보자.  
  
---
 
### github workflow  
  
  
``` yaml
name: Deploy to EC2

on:
  push:
    branches: ["release"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - name: Build (Gradle)
        run: ./gradlew clean build -x test

      - name: Upload JAR to EC2
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SECRET }}
          source: "build/libs/*.jar"
          target: "/home/ubuntu/app/ticketpark"
          strip_components: 2

      - name: Restart on EC2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SECRET }}
          script: |
            cd /home/ubuntu/app/ticketpark
            mv *.jar app.jar
            sudo systemctl restart ticketpark.service
```  
  
위와 같이 세팅하고 Github Action에서 외부로 안보여야 하는 환경변수를 설정해주면  
아까 만들어놓은 ~/app/ticketpark에 app.jar파일로 생성된다.  
  
이때 자바파일이 SNAPSHOT-plain.jar 와 SNAPSHOT.jar 이렇게 두 개가 생성이 되는데 이것은 build.gradle에서  
```gradle
jar {
	enabled = false
}
```
이렇게 옵션을 추가하면 단독으로 실행이 가능한 빌드파일만 나온다.  
  
Ubuntu에서 systemd에 서비스를 데몬으로 넣어주고 릴리즈 브랜치로 푸쉬를 하면  
자동배포 설정이 끝난다.  

---  

### 배포 성공

가끔은 몸이 기억하는 명령어가 많다.  
나한테는 리눅스 명령어들이 유독 그렇다.  
  
로그 보는 명령어가 있었던 거 같은데... 하고 물 한잔 마시고 와서 아무 생각 없이 검은 화면에 초록글자를 보니  
자연스럽게 나왔다...  
**journalctl -xefu ticketpark.service**  
  
분명 기억이 안났는데 났다.

<img src="/assets/images/posts/largeTrafficService/yeah.png" width="100%" height="50%">  
~~웅장해진다...!~~  

현재는 EC2 내부에 Redis와 Mysql을 모두 설치해놓고 테스트를 진행할 것이다.  
이제 나의 서버가 생겼으니 부하테스트를 해보도록 하자.