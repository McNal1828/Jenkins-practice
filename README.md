# Jenkins 를 사용한 pipeline 구축

Jenkins를 설치하고 github를 사용하여 CI/CD pipeline을 구축한다.

# 구성

- VM
	- ubuntu-live-server-22.04
	- 2 cpu
	- 4GB RAM
	- 32GB Disk
	
> Jenkins 최소권장사양인 4GB RAM을 할당한다

# 설치

## Jenkins

[공식홈페이지의 문서](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)를 참고하여 설치를 진행한다.

> Jenkins 2.452.2(2024.06.14)버전입니다

### Jenkins LTS release

```sh
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

### JAVA

```sh
sudo apt update
sudo apt install fontconfig openjdk-17-jre
```

### 설치 확인 및 실행

```sh
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

기본포트는 `8080`번이며 초기 `Administrator password`는 `/var/lib/jenkins/secrets/initialAdminPassword`의 log로 확인가능하다

> 시스템에 따라서 log의 위치는 변경될 수 있다.
> 웹 브라우저에 나타나는 주소를 확인한다.

이후 초기세팅을 마무리한다.