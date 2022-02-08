---
title: Github로 자바 라이브러리 배포하기
author: keencho
date: 2022-01-22 20:12:00 +0900
categories: [Java]
tags: [Java, Github, Maven]
---

# **개요**
사이드 프로젝트를 진행하다보면 최초 세팅시 공통된 코드가 사용되는 경우가 많습니다. 하지만 이를 모듈화하여 개인 `nexus`에 배포하자니 서버 구축이 어렵고, `Maven Central`에 배포하자니 거쳐야하는 단계가 많아 어려움을 겪었었습니다.

이 포스팅 에서는 Github를 이용해 개인 라이브러리를 배포하는 몇가지 방법에 대해 알아보겠습니다.

> 라이브러리 배포에 있어서는 Maven이 Gradle 보다 용이하다고 생각되어 아래 내용은 모두 Maven을 기반으로 작성되었습니다. (저도 개인적으로는 Gradle을 선호합니다...)

아래 예제는 [제가 쓰려고 만든 라이브러리](https://github.com/keencho/lib-custom-p6spy) 를 배포하는 과정을 담고 있습니다.

## **1. Github Repository를 Maven Repository 처럼 사용하기**
첫번째 방법은 Github repository를 maven repository처럼 사용하는 방법입니다.

### **1.1. Local & Github Repository 만들기**

##### **1.1.1. github에 public repository 생성하기**
빈 repository를 하나 생성합니다. 이름은 임의로 지정하였습니다.
![init-repo](/assets/img/custom/github/custom-maven-repository/init-repo.jpg)

##### **1.1.2. 방금 생성한 repository의 이름으로 폴더 생성하기**
```
mkdir maven-test-repo
```

##### **1.1.3. 폴더를 git repository로 만들고 github에서 생성한 repository와 연결**
```
cd maven-test-repo
git init
git remote add origin https://github.com/keencho/maven-test-repo.git
```

##### **1.1.4. releases, snapshots 폴더 생성**
```
mkdir releases, snapshots
```

### **1.2. Local에 내 라이브러리 배포하기 & Github Repository에 배포하기**

##### **1.2.1. 배포할 라이브러리의 pom.xml 수정하기**

```xml

 ...

 <groupId>com.keencho.lib</groupId>
 <artifactId>keencho-p6spy</artifactId>
 <version>1.0.1</version>

 ...

 <distributionManagement>
     <repository>
         <id>release</id>
         <url>https://github.com/keencho/maven-test-repo/raw/master/releases</url>
     </repository>

     <snapshotRepository>
         <id>snapshot</id>
         <url>https://github.com/keencho/maven-test-repo/raw/master/snapshots</url>
     </snapshotRepository>
 </distributionManagement>
```
이때 groupId, artifactId, version 태그는 추후 어플리케이션에 의존성 추가할때 쓰임을 인지하고 이름을 정해주세요.

##### **1.2.2. altDeploymentRepository 명령어로 라이브러리 로컬에 배포하기**

altDeploymentRepository 명령어로 라이브러리를 로컬에 배포합니다.

```
mvn -D altDeploymentRepository=snapshot::default::file:../maven-test-repo/snapshots clean deploy
```

파일 구조는 다음과 같습니다.
 ```
 └── D:
     └── workspace
         ├── maven-test-repo
         │   ├── releases
         │   └── snapshots (배포될 디렉토리)
         └── keencho-lib-custom-p6spy (배포할 라이브러리)
 ```

##### **1.2.3. 배포 확인후 github 에 push 하기**
local 폴더에 정상적으로 배포되었다면 commit - push를 진행합니다.

### **1.3. 라이브러리를 사용할 어플리케이션에 의존성 추가**

##### **1.3.1. 라이브러리를 사용할 어플리케이션에 repository 연결정보를 추가**
```xml
<distributionManagement>
    <repository>
        <id>release</id>
        <url>https://github.com/keencho/maven-test-repo/raw/master/releases</url>
    </repository>

    <snapshotRepository>
        <id>snapshot</id>
        <url>https://github.com/keencho/maven-test-repo/raw/master/snapshots</url>
    </snapshotRepository>
</distributionManagement>
```

##### **1.3.2. 의존성 추가**
```xml
<dependency>
    <groupId>com.keencho.lib</groupId>
    <artifactId>keencho-p6spy</artifactId>
    <version>1.0.1</version>
</dependency>
```
위에서 정의한 groupId, artifactId, version 을 동일하게 입력합니다.

##### **1.3.3. 확인하기**
명령어로 maven install 후 내가 만든 라이브러리가 추가되어있는지 확인합니다.

```
mvn install
```
![custom-repo-check](/assets/img/custom/github/custom-maven-repository/custom-repo-check.jpg)

## **2. Github Packages 사용하기**
두번째 방법은 Github Packages 를 사용하는 방법입니다.

> 주의! 이 방법은 `GITHUB_TOKEN` 을 사용하는 방법입니다. 배포할때 뿐만 아니라 의존성을 추가할때도 토큰이 필요합니다.
> 이에대해 [github 개발자들도 인지하고 있고](https://github.community/t/how-to-allow-unauthorised-read-access-to-github-packages-maven-repository/115517/3) 21년 말까지 public access를 허용하겠다고 하였으나 22년 초 현재까지 소식이 없습니다.

### **2.1. Github Packages 에 인증하기**

##### **2.1.1. Personal Access Token 발급받기**
첫번째로 github token을 발급 받아야 합니다. 토큰 토큰 발급 과정은 다음과 같습니다.
```
github -> 우측 상단 프로필 -> Settings -> 좌측 사이드바 최하단 Developer settings -> 좌측 사이드바 Personal access tokens -> Generate new token
```

![github-token](/assets/img/custom/github/custom-maven-repository/github-token.png)

임의의 이름을 입력하고 만료일을 선택한후, 스코프를 지정합니다. 원래대로라면 엄격하게 지정해야 하지만 테스트후 바로 삭제할 것이므로 모든 스코프에 체크한후 토큰을 생성합니다.

##### **2.1.2. 발급받은 토큰으로 settings.xml 세팅하기**
발급받은 토큰을 기반으로 settings.xml 파일을 세팅하면 `GitHub Packages with Apache Maven`에 인증할 수 있습니다.

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      http://maven.apache.org/xsd/settings-1.0.0.xsd">

  <activeProfiles>
    <activeProfile>github</activeProfile>
  </activeProfiles>

  <profiles>
    <profile>
      <id>github</id>
      <repositories>
        <repository>
          <id>central</id>
          <url>https://repo1.maven.org/maven2</url>
        </repository>
        <repository>
          <id>github</id>
          <url>https://maven.pkg.github.com/keencho/lib-custom-p6spy</url>
          <snapshots>
            <enabled>true</enabled>
          </snapshots>
        </repository>
      </repositories>
    </profile>
  </profiles>

  <servers>
    <server>
      <id>github</id>
      <username>keencho</username>
      <password>{TOKEN}</password>
    </server>
  </servers>
</settings>
```

### **2.2. 라이브러리 배포하기**
기본적으로 Github는 패키지와 동일한 이름의 기존 레포지토리에 패키지를 배포합니다. 예를들어 `com.example:test` 패키지는 `OWNER/test` 라는 리포지토리에 배포될 것입니다.

##### **2.2.1. pom.xml 수정하기**
배포할 라이브러리의 pom.xml을 수정합니다. `OWNER` 은 사용자 계정의 이름이며 `REPOSITORY`는 배포할 라이브러리의 패키지명 입니다. (여기서는 artifactId)

```xml
<distributionManagement>
   <repository>
     <id>github</id>
     <name>keencho</name>
     <url>https://maven.pkg.github.com/keencho/lib-custom-p6spy</url>
   </repository>
</distributionManagement>
```

##### **2.2.2. 배포하기**
```
mvn deploy
```
위 명령어로 배포합니다.

![github-package-deploy](/assets/img/custom/github/custom-maven-repository/github-package-deploy.png)
위 캡쳐화면과 같이 github maven 저장소에 업로드 되었다는 로그를 확인할 수 있습니다.

##### **2.2.3. Github에서 확인하기**
Github UI 화면에서도 확인할 수 있습니다. 리포지토리의 우측 사이드바를 확인해보세요.
![github-package-1](/assets/img/custom/github/custom-maven-repository/github-package-1.png)
![github-package-2](/assets/img/custom/github/custom-maven-repository/github-package-2.png)

### **2.3. 라이브러리 설치하기**

`Github Packages` 에 인증되어 있어야 라이브러리 설치가 가능합니다. 여기까지 진행하셨다면 당연히 인증되어 있겠지만 혹시 인증되어 있지 않다면 `2.1.` 항목의 과정들이 선행 되어야 합니다.

##### **2.3.1. 의존성 추가 후 설치하기**
방금 배포한 라이브러리의 의존성을 추가합니다.

```xml
<dependencies>
  <dependency>
    <groupId>com.keencho.lib</groupId>
    <artifactId>keencho-p6spy</artifactId>
    <version>1.0.1</version>
  </dependency>
</dependencies>
```

의존성을 추가하였다면 아래 명령어를 통해 설치합니다.
```
mvn install
```

설치가 완료되었다면 라이브러리가 추가되었는지 확인합니다.

## **3. Jitpack으로 배포하기**
마지막 세번째 방법은 Jitpack을 활용해 배포하는 방법입니다. Jitpack을 사용하면 별도 설정 없이 (물론 Jitpack 에 빌드되어 배포는 되어 있어야 합니다.) Github 저장소의 주소를 원격 저장소로 사용할 수 있습니다.

### **3.1. Jitpack 살펴보기**

##### **3.1.1. 로그인하기**
[Jitpack](https://jitpack.io/)에 접속해 우측 상단 버튼을 클릭해 Github로 로그인합니다.
![jitpack-login](/assets/img/custom/github/custom-maven-repository/jitpack-login.png)

##### **3.1.2. 리포지토리들 확인하기**
좌측 사이드바에 내 리포지토리들이 잘 보이는지 확인합니다.
![jitpack-repository](/assets/img/custom/github/custom-maven-repository/jitpack-repository.png)

### **3.2. Github Release**

##### **3.2.1 JDK 세팅**
jitpack은 빌드시 jdk8을 사용합니다. 만약 라이브러리의 타겟 jdk가 11이라면 기본 설정으로는 빌드되지 않습니다.

원하는 jdk로 빌드 되도록 하기 위해 `jitpack.yml` 파일을 프로젝트 루트폴더에 생성하고 다음과 같이 사용할 jdk를 명시합니다.
```yaml
jdk: openjdk11
```

##### **3.2.2. code push**
릴리즈를 생성하기 전에, 작업 결과물을 commit & push 합니다.

##### **3.2.3. release**
Github release 기능을 이용하여 릴리즈를 생성 합니다.

![github-release-1](/assets/img/custom/github/custom-maven-repository/github-release-1.png)
![github-release-2](/assets/img/custom/github/custom-maven-repository/github-release-2.png)
![github-release-3](/assets/img/custom/github/custom-maven-repository/github-release-3.png)

위 캡쳐화면의 플로우로 릴리즈 생성 페이지에 들어와 tag를 지정하고 `publish relase` 버튼을 눌러 배포합니다.

여기서 tag는 추후 maven의 version이 되기 때문에 필수이며 title과 describe는 필수 입력 항목이 아닙니다.

##### **3.2.4. release 확인**
릴리즈가 완료되었음을 확인합니다.
![github-release-4](/assets/img/custom/github/custom-maven-repository/github-release-4.png)

### **3.3. Jitpack 배포**

##### **3.3.1. Look up**
Jitpack 배포를 수행하기 위한 첫번째 단계로 `Look up` 을 진행해야 합니다. Jitpack에 접속후 `Owner/Repository` 의 양식으로 인풋 박스에 입력후 `Look up` 버튼을 클릭합니다.
![jitpack-lookup](/assets/img/custom/github/custom-maven-repository/jitpack-lookup.png)

그럼 위와같은 화면이 보이실 텐데요, 저는 이미 배포가 완료된 상태라 `Get it` 버튼이 초록색이지만 아직 배포되지 않은 경우 버튼의 색상이 하얀색으로 보이게 됩니다.

하얀색 `Get it` 버튼을 눌러 배포를 진행합니다.

##### **3.3.2 Jitpack 배포 확인**
`Log` 버튼을 눌러 배포가 완료되었는지 확인합니다. 프로세스가 끝나지 않은 경우 파일이 열리지 않으니 파일이 열리지 않는 경우 조금 기다려 보세요.

파일이 열렸다면 스크롤을 최하단으로 내려 아래와 같이 `BUILD SUCCESS` 로그가 찍혀있는지 확인해 보세요.

```
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  6.712 s
[INFO] Finished at: 2022-02-08T04:59:37Z
[INFO] ------------------------------------------------------------------------
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8 -Dhttps.protocols=TLSv1.2
Found module: com.keencho.lib:keencho-p6spy:1.0.2
Build tool exit code: 0
Looking for artifacts...
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8 -Dhttps.protocols=TLSv1.2
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8 -Dhttps.protocols=TLSv1.2
Looking for pom.xml in build directory and ~/.m2
Found artifact: com.keencho.lib:keencho-p6spy:1.0.2
Exit code: 0

✅ Build artifacts:
com.github.keencho:lib-custom-p6spy:1.0.2

Files:
com/github/keencho/lib-custom-p6spy/1.0.2
com/github/keencho/lib-custom-p6spy/1.0.2/build.log
com/github/keencho/lib-custom-p6spy/1.0.2/lib-custom-p6spy-1.0.2-sources.jar
com/github/keencho/lib-custom-p6spy/1.0.2/lib-custom-p6spy-1.0.2.jar
com/github/keencho/lib-custom-p6spy/1.0.2/lib-custom-p6spy-1.0.2.jar.md5
com/github/keencho/lib-custom-p6spy/1.0.2/lib-custom-p6spy-1.0.2.jar.sha1
com/github/keencho/lib-custom-p6spy/1.0.2/lib-custom-p6spy-1.0.2.pom
com/github/keencho/lib-custom-p6spy/1.0.2/lib-custom-p6spy-1.0.2.pom.md5
com/github/keencho/lib-custom-p6spy/1.0.2/lib-custom-p6spy-1.0.2.pom.sha1
```

### **3.4. 최종 확인**

##### **3.4.1. 의존성 추가 후 설치하기**
Jitpack에 배포가 완료되었다면 이제 의존성 추가 후 사용하는 일만 남았습니다.

pom.xml에 아래와 같이 태그를 추가해 주세요. `jitpack.io`를 리포지토리로 사용하겠다는 태그도 추가해 줘야 합니다.

```xml
<repositories>
    <repository>
        <id>jitpack.io</id>
        <url>https://jitpack.io</url>
    </repository>
</repositories>

...

<dependency>
    <groupId>com.github.OWNER</groupId>
    <artifactId>REPOSITORY</artifactId>
    <version>TAG</version>
</dependency>
```

이때 `OWNER`은 github 계정 이름, `REPOSITORY`는 리포지토리명, `TAG`는 github relase 생성시 입력한 tag 입니다. 저의 경우는 다음과 같습니다.

```xml
<dependency>
    <groupId>com.github.keencho</groupId>
    <artifactId>lib-custom-p6spy</artifactId>
    <version>1.0.2</version>
</dependency>
```

세팅이 모두 완료되었다면 mvn install 명령어를 통해 설치후 라이브러리가 추가되어 있는지 확인해 보세요.
```
mvn install
```

### **참조**
- [https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-apache-maven-registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-apache-maven-registry)
- [https://jitpack.io/docs/BUILDING/](https://jitpack.io/docs/BUILDING/)
