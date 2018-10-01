# Maven SSH Deploy

## 1. 배포서버 설정
- 참고 레퍼런스 : https://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html
- mave ssh 배포 : https://blog.csdn.net/xiao__gui/article/details/49420813

- jar 경로 : /home/ec2-user/app/myapp.jar
- log 경로 : /home/ec2-user/app/logs

- conf 파일 (java 설정)
```
# vi /home/ec2-user/app/myapp.conf

JAVA_HOME=/usr/local/jdk-11
JAVA_OPTS="-Dspring.profiles.active=dev -Xms256m -Xmx256m -verbose:gc -Xloggc:/home/ec2-user/app/logs/gc.log -XX:+PrintGCDetails"
```

- systemctl service 파일 추가
```
# sudo vi /etc/systemd/system/myapp.service
[Unit]
Description=Spring boot application
After=network.target

[Service]
SyslogIdentifier=SpringBootApplication
ExecStart=/home/ec2-user/app/myapp.jar
WorkingDirectory=/home/ec2-user/app
User=ec2-user
Type=simple
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target

설정 적용
# sudo systemctl daemon-reload

시작|정지|재시작
# systemctl start|stop|restart myapp

jar 파일 실행 후 서버가 떠 있는지 확인
# ps -ef|grep java
```

- Tip) 만약 myapp.jar 을 ls -al 로 확인했을 경우 실행(x) 권한이 없으면 chmod +x myapp.jar 로 실행권한을 준다.


## 2. maven 환경설정 파일에 ec2 로그인 정보를 추가.
- IntelliJ Project -> 마우스 우클릭 -> Maven -> Open 'settings.xml' or Create 'settings.xml' 메뉴를 이용하면 편리.
- Windows : C:\Users\{username}\.m2\settings.xml
- Mac : /Users/{username}/.m2/settings.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <servers>
        <server>
            <id>devserver</id> <!-- 서비스ID : maven에서 인식시킬 고유 아이디 -->
            <username>ec2-user</username> <!-- ssh 계정 -->

            <!-- privateKey 로 로그인 할 경우 -->
            <privateKey>mykey.ppk</privateKey> <!-- private key (.ppk) 파일 경로) -->
            <passphrase>privateKey password</passphrase> <!-- ppk 생성시 비밀번호를 입력하였을 경우 입력, 안했으면 공백 -->

            <!-- ssh password 로 로그인 할 경우 위의 privateKey, passphrase 는 제거 -->
            <password>password</password> <!-- ssh 패스워드 -->

            <!-- Windows 의 경우 configuration ssh, scp 명령어 alias 추가, Mac 은 주석 처리 -->
            <!-- putty : https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html -->
            <!-- putty 64bit installer 로 설치 -->
            <configuration>
                <sshExecutable>plink</sshExecutable>
                <scpExecutable>pscp</scpExecutable>
            </configuration>
        </server>
    </servers>
</settings>

3. pom.xml 설정

    <build>
        <finalName>myapp.jar</finalName> <!-- 빌드되는 파일명 지정 -->

        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <executable>true</executable> <!-- executable jar 설정 추가 -->
                </configuration>
            </plugin>
        </plugins>

        <!-- ssh maven extension library 추가 -->
        <extensions>
            <extension>
                <groupId>org.apache.maven.wagon</groupId>
                <!-- settings.xml 에서 설정한 configuration 으로 ssh, scp 명령어 alias 를 가능하게 해주는 라이브러리 -->
                <!-- wagon-ssh 라이브러리는 alias 불가능한 라이브러리 임 -->
                <artifactId>wagon-ssh-external</artifactId>
                <version>3.1.0</version>
            </extension>
        </extensions>
    </build>

    <!-- profile 별로 배포를 다르게 구성하기 위해 dev profile 추가 -->
    <profiles>
        <profile>
            <id>dev</id>
            <build>
                <plugins>
                    <!-- jar 파일을 ssh 로 업로드 하기 위한 plugin -->
                    <plugin>
                        <groupId>org.codehaus.mojo</groupId>
                        <artifactId>wagon-maven-plugin</artifactId>
                        <version>2.0.0</version>
                        <executions>
                            <execution>
                                <id>upload-single-jar</id>
                                <phase>package</phase> <!-- package 단계에서 실행하도록 -->
                                <goals>
                                    <goal>upload-single</goal> <!-- single file upload -->
                                </goals>
                                <configuration>
                                    <serverId>devserver</serverId> <!-- settings.xml 에서 설정한 서비스ID -->
                                    <url>scpexe://127.0.0.1:/home/ec2/app/myapp.jar</url>
                                    <fromFile>${project.build.directory}/myapp.jar</fromFile> <!-- 빌드로 생성된 jar 파일명 -->
                                    <toFile>myapp.jar</toFile>
                                </configuration>
                            </execution>
                            <execution>
                                <id>sshexec-restart</id>
                                <phase>package</phase>
                                <goals>
                                    <goal>sshexec</goal> <!-- run shell script -->
                                </goals>
                                <configuration>
                                    <serverId>devserver</serverId> <!-- settings.xml 에서 설정한 서비스ID -->
                                    <url>scpexe://127.0.0.1:/home/ec2/app</url>
                                    <displayCommandOutputs>true</displayCommandOutputs>
                                    <failOnError>false</failOnError>
                                    <commands>
                                        <command>sudo systemctl restart myapp.service</command>
                                    </commands>
                                </configuration>
                            </execution>
                        </executions>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
```

## 3. 배포 명령어
```
# mvnw clean package -Pdev
```