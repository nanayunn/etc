# Ansible



## 1. Ansible이란?

* 여러 개의 서버를 효율적으로 관리하기 위해 고안된 *환경 구성 자동화 도구*
  * remote 서버에 접속해서 무언가를 시행
  * 고급 단계
    * 모니터링하는 복잡한 환경에서 LB를 사용할 수 있도록 함.

* 2012년 출시, Redhat에 인수되어 개발
* Python 기반의 오픈소스, 
* yaml 문법으로 정의
* json으로 통신



*  **Infrastructure as Code**을 지향하는 자동화 관리도구

  > **IaC( Infrastructure as Code )**란?
  >
  > * 기존 스크립트 또는 수동의 CLI기반 시스템 프로비저닝 방식에서 탈피,
  > * 프로그래밍 코드 기반으로 시스템 자동 설치 및 구축, 관리, 프로비저닝을 구현하는 IT 인프라 구성 프로세스
  >   * 프로그래밍이 가능한 인프라로 정의되는 이유

  

* 인프라의 상태를 코드로 선언하고 이를 모든 서버에 배포함으로써 특정 환경을 동일하게 유지할 수 있도록 돕는다.



* 사용되는 측면 : 
  * 환경의 배포
  * 서버 클러스터의 체계적인 관리
  * 확장 가능한 모듈의 사용 등등..



## 2. Ansible 장점

#### 1. Agentless

* 대부분의 IaC 도구 : Agent 기반으로 구성하는 Pull 방식
* **Ansible** : Agentless 기반의 Push 방식



* 대상 장비에 Agent를 설치 하지 않는다!
  * IT 인프라 담당자들의 거부감 ↓
* SSH 기반의 동작
  * 기술적 접근성 용이
* 한정된 서버 영역이 아닌, 폭넓은 영역에서 사용될 수 있다.



#### 2. 접근 용이성

* Python 기반으로 제작되었지만 언어 지식 없이도 사용가능
* Playbook
  * YAML 사용
    * 가독성이 좋으므로 DevOps 진입이 쉽다.



#### 3. 다양한 모듈 지원

* 지속적인 패치와 접근 용이성으로 인한 넓은 사용자층
  * 많은 모듈들을 지원



#### 4. 멱등성( idempotence )

>  멱등성이란?
>
> * 연산을 여러번 수행하더라도 결과가 달라지지 않는 성질

* YMAL로 작성된 Playbook을 여러차례 반복하더라도 동일한 결과 출력
* 자동화 도구 실행시 발생할 수 있는 오동작, 설정오류 등의 문제 해소

* 다만, shell, command, file module은 보장이 안된다.



## 3. Ansible의 한계

* 시스템의 초기 설치 수행은 불가능	
  * kickstart, pxe 등을 사용해야함
* 시스템 모니터링은 지원하지 않음
* 시스템 변경사항은 추적하지 않음



## 4. Ansible 구성요소

![img](https://blog.kakaocdn.net/dn/cuOWvb/btqCDkueHqy/MHnXKUDOig9gHdkuQNiFD0/img.png)



* Inventory : 

  > ‘어디에서’ 앤서블을 실행하는가

  * 관리되는 노드의 목록( 원격 서버들의 목록 )
  * 호스트 파일이라고도 한다. ( /etc/ansible/hosts 파일 )
    * 인벤토리는 관리되는 노드에 대한 각각의 IP주소와 같은 정보를 지정
    *  관리 노드를 구성하여 쉽게 확장 가능하도록 그룹을 만들고 중첩시킬 수 있다. 

* Modules :

  > ‘무엇을’ 앤서블에서 실행하는가

  * host에 특정 action을 수행하는 패키지화된 scripts
  * 앤서블에서 실행된 하나하나의 명령( Ansible 코드의 실행 단위 )
    - 예) 패키지 설치와 서비스, 파일 복사와 편집, MySQL 과 같은 사용자 테이블 관리, AWS, 애저 서비스 작업 등등
    - 셸 스크립트로 작성하면 복잡한 절차가 필요하지만, 앤서블에 내장된 모듈은 사용자가 의식하지 않고도 안전하고 적절한 처리를 실행할 수 있도록 함.
  * 내장 모듈 목록
    * : http://docs.ansible.com/ansible/devel/modules/modules_by_category.html
  * 자주 사용하는 모듈
    - 파일 작업
      - file : 파일 작업
      - copy : 파일을 작성 대상에게 전송
      - lineinfile : 기존 파일을 행 단위로 수정
    - 명령어 실행 모듈
      - command

* Play-book : 

  > ‘어떻게’ 앤서블을 실행하는가

  * 원격 서버에 전달할 명령들을 모아둔 명령집

  * 변수 및 task를 관리 호스트에 수행하기 위해 yaml 문법으로 정의된 파일
  * 반복해서 실행하고자 해당 작업을 실행 순서대로 저장해 놓은 정렬된 작업 리스트

* Task:

  * Ansible의 작업 단위
  * `ad-hoc` 명령을 사용하여 단일 작업을 한 번 실행 가능

* plug-in

  * 확장 기능(email, logging, etc)을 제공

* Custom module

  * 사용자가 직접 작성한 모듈을 사용





## 5. Ubuntu에 Ansible 설치하기

```
$ sudo apt update
$ sudo apt install software-properties-common
$ sudo apt-add-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible
```

**기본적인 Ansible command 또는 playbook**

- 인벤토리에서 실행할 기계를 선택한다.

  - Ansible은 인벤토리에서 관리하려는 머신에 대한 정보를 읽는다.

- 일반적으로 SSH를 통해 해당 시스템 (or 네트워크 장치 or 기타 관리 노드)에 연결한다.

- 하나 이상의 모듈을 원격 시스템에 복사하고, 거기서 실행을 시작한다.





## 6. Ansible test

* `ansible localhost -m ping` 테스트
* `ansible localhost -a "/bin/echo hello" 테스트`



### VirtualBox 가상 서버와 로컬 서버로 ansible 실습



#### 1. ping 테스트



**가상 서버**

1. 호스트(로컬)과 통신하기 위해 호스트 어댑터를 이용 
   * enp0s8 netplan 설정으로 IP를 할당해야함.
   * ip 목록 확인

<img src="https://user-images.githubusercontent.com/58680504/88760644-707a5e00-d1a8-11ea-9e7b-a9d10b441883.png" alt="스크린샷, 2020-07-29 13-54-39"  />



2. enp0s3의  /etc/netplan/50-clound-init.yml 파일 내용

![스크린샷, 2020-07-29 13-55-06](https://user-images.githubusercontent.com/58680504/88760645-7112f480-d1a8-11ea-9a29-a063d31e0403.png)



3. enp0s8을 위해 새롭게 /etc/netplan/01-netcfg.yml 파일 생성,
   * 아래의 내용을 작성. 
   * 인터페이스에 적용한 호스트 어댑터 gateway는 ( 192.168.56.1 /24 )  

![스크린샷, 2020-07-29 13-55-28](https://user-images.githubusercontent.com/58680504/88760647-7112f480-d1a8-11ea-976e-1b6213aad9f4.png)



4. 파일을 생성, 내용을 모두 작성한 후 다음 명령어를 입력

![스크린샷, 2020-07-29 13-55-50](https://user-images.githubusercontent.com/58680504/88760649-71ab8b00-d1a8-11ea-94cc-9009f9a59ee5.png)



5. `ifconfig -a` 로 enp0s8 인터페이스에 ip가 할당된 것을 볼 수 있다.

![스크린샷, 2020-07-29 13-56-09](https://user-images.githubusercontent.com/58680504/88760650-71ab8b00-d1a8-11ea-8bf5-46500aafc643.png)



6. 가상서버에도 `sudo apt-get install ssh`, 혹은 `sudo apt-get install openssh-server`을 설치해 준다.
   * 나는 `.ssh` 폴더를 삭제해버려서.. 다시.. 만듦..

![스크린샷, 2020-07-29 13-56-51](https://user-images.githubusercontent.com/58680504/88760651-72442180-d1a8-11ea-9ef9-336935edd93b.png)



**로컬**

7. 로컬 서버에서 ssh 설치 후, 퍼블릭 키 생성

![스크린샷, 2020-07-29 13-51-29](https://user-images.githubusercontent.com/58680504/88760635-6e180400-d1a8-11ea-88b5-1c831907b57e.png)



8. 퍼블릭 키 생성 확인

![스크린샷, 2020-07-29 13-51-48](https://user-images.githubusercontent.com/58680504/88760638-6eb09a80-d1a8-11ea-96db-6e7f1f5574dc.png)



9. key를 가상서버에 복사한다.

![스크린샷, 2020-07-29 13-53-22](https://user-images.githubusercontent.com/58680504/88760640-6f493100-d1a8-11ea-8caf-215f11ef266b.png)



10. 이후, `/etc/ansible/hosts` 파일을 `sudo` 권한으로 열어서 가상서버의 ip `192.168.56.100`을 적어준다.



**가상 서버**

11. `~/` 위치로 가면 ssh 키가 복사되어 있는 것을 볼 수 있다. 키를 `~/.ssh/authorized_keys`로 복사한다.

![스크린샷, 2020-07-29 13-57-03](https://user-images.githubusercontent.com/58680504/88760653-72dcb800-d1a8-11ea-95ec-b0967b5beb94.png)



**로컬**

12. 그냥 실행 시 `root` 의 계정으로 실행이 되기 때문에 그냥하면 실패한다.
    * `--user=<user id>` 옵션을 붙여주면 제대로 ping 모듈이 실행되는 것을 볼 수 있다.

![스크린샷, 2020-07-29 13-53-38](https://user-images.githubusercontent.com/58680504/88760641-6fe1c780-d1a8-11ea-8619-690f62e59b52.png)
![스크린샷, 2020-07-29 13-53-47](https://user-images.githubusercontent.com/58680504/88760643-707a5e00-d1a8-11ea-8839-a08cf350ff4e.png)





#### 2. playbook 작성을 통한 테스트

##### 2-1. ping 테스트

13. `YAML` 파일 작성이 `sudo` 권한으로만 된다...

![스크린샷, 2020-07-29 14-10-00](https://user-images.githubusercontent.com/58680504/88760656-73754e80-d1a8-11ea-99d3-b06e6eadc4e0.png)



14. `test.yml` 파일에는 다음과 같이 적었다.

![스크린샷, 2020-07-29 14-10-50](https://user-images.githubusercontent.com/58680504/88760663-74a67b80-d1a8-11ea-84e0-0347d9b38eab.png)

(처음에 `hosts` 설정값을 가상서버의 서버이름`controller`로 작성하였더니 실행이 안되는 부분)

![스크린샷, 2020-07-29 14-10-28](https://user-images.githubusercontent.com/58680504/88760658-740de500-d1a8-11ea-8deb-151f77e8895a.png)



15. `/etc/ansible/hosts` 파일에 적은 가상서버의 IP 값을 적고 실행하면 제대로 실행된다.

![스크린샷, 2020-07-29 14-10-39](https://user-images.githubusercontent.com/58680504/88760660-740de500-d1a8-11ea-94bf-381f809bcbda.png)



##### 2-2. Create User 테스트



16. 이번에는 `test.yml` 파일 내용을 다음과 같이 바꾼다.

![스크린샷, 2020-07-29 14-13-42](https://user-images.githubusercontent.com/58680504/88760664-74a67b80-d1a8-11ea-9fe5-65f9a7237b4d.png)



17. 그냥 실행했더니 권한이 없어서 실행이 안된다.( Permission Denied )

![스크린샷, 2020-07-29 14-14-14](https://user-images.githubusercontent.com/58680504/88760665-753f1200-d1a8-11ea-90ac-2c9b58afe27d.png)





18. `become: yes` 옵션으로 해당 task 실행에 `sudo` 권한을 주었다.

![스크린샷, 2020-07-29 14-15-20](https://user-images.githubusercontent.com/58680504/88760667-75d7a880-d1a8-11ea-8dd8-4e3db7824271.png)



19. `-K` 옵션을 주면 `sudo` 권한 부여에 대한 패스워드를 치게 한다.
    * 새로운 유저가 생성되었으므로 `changed = 1`이 표시된 값을 볼 수 있다.

![스크린샷, 2020-07-29 14-17-05](https://user-images.githubusercontent.com/58680504/88760668-75d7a880-d1a8-11ea-8b7a-1578186da9c4.png)



20. 진짜 생성되었는지 보기 위해서 해당 명령어 실행
    * `/etc/passwd` : 서버 내에 사용자 목록이 적혀있는 파일

![스크린샷, 2020-07-29 14-22-54](https://user-images.githubusercontent.com/58680504/88760670-7708d580-d1a8-11ea-98db-9bd3e75e6b10.png)

21. `ansible_user`가 생성되어 있는 것을 볼 수 있다. 

![스크린샷, 2020-07-29 14-22-39](https://user-images.githubusercontent.com/58680504/88760669-7708d580-d1a8-11ea-9368-9cea27d9b2f5.png)







## 7. Ansible test2( Ansible과 Docker로 실습  )

* 환경
    * VirtualBox 가상 서버
    * 설치 및 활용 프로그램
        * Vagrant, Docker, Ansible
 * 테스트 목표
     * 로컬 서버와 가상서버로 진행했던 playbook 테스트를 가상서버와 여러 개의 도커 컨테이너에서 진행해보기



* 참고 사이트 : 
  * https://dev-yakuza.github.io/ko/environment/vagrant-install-and-usage/



#### 테스트 진행 순서

#### 1. VirtualBox

* 6.1 최신버전으로 재설치

* `sudo apt-get remove virtualbox`

  * 의존성 오류 발생 시:
    * `sudo apt --fix-broken install`

* `sudo dpkg -i virtualbox 6.1 deb 패키지`

  * 필요시 `sudo apt install gdebi`
  * `sudo apt-get -f install`

  

#### 2. Vagrant

* **Vagrant 초기화하고 싶다면**
  * vagrant 실행 파일이 존재하는 곳에 `.vagrant`가 있는지 확인 및 삭제
  * `/opt/vagrant`가 있는지 확인 및 삭제
  * `/usr/bin/vagrant`가 있는지 확인 및 삭제
  * `~/vagrant.d`가 있는지 확인 및 삭제( `sudo` )

* Linux는 zip 파일로 존재

  * `unzip` 실행 시 실행 프로그램 번들이 나오는데, 설치 방법을 알아내지 못함.
    * `./vagrant`로 프로그램 실행

* vagrant 전용 폴더를 생성하여 진행함

  * `mkdir vagrant`

* `./vagrant init`

* 공식 box와 user box를 추가해야한다.

  * 공식 box 사이트: https://app.vagrantup.com/boxes/search
  * 유저 box 사이트: http://www.vagrantbox.es/

* 공식 box 추가

  * `./vagrant box add ubuntu/bionic64`

* user box 추가

  * `./vagrant box add bento/ubuntu 16.04`

  > * 생성된 box 확인
  >   * `./vagrant box list`
  > * 생성된 box 삭제
  >   * `./vagrant remove box bento/ubuntu-16.04`

* 가상 머신 생성

  * `./vagrant up`

* 가상 머신 접속

  * `./vagrant ssh`
  * 로그아웃 : `exit`



* vagrant를 ansible playbook으로 설정 관리했을 때 가장 해결하기 어려웠던 오류..
  * python-apt에 대한 dependency 오류,
  * 이로 인한 apt cache update 불가 오류
  * python3.6 버전 설치를 위해 PPA repository를 추가하여 설치를 시도하였으나
    * 1. 릴리즈 버전 파일이 해당 사이트에 없으며, 접근 불가능하다는 오류
      2. 실제로 페이지에 가니 상업적 용도를 막기위해 모든 파일을 public 접근을 막았다는 글을 볼 수 있었다. 
      3. ppa:deadsnakes/ppa 경로에는 파일이 존재하지만, 
         * ansible playbook 내에서는 계속해서 경로를 찾지 못하고 실패하였음
      4. ansible playbook이 아니라 vagrant 설정 파일 아래에 직접 명령어로 넣어줬다.
         * 설치를 위해서는 `add-apt-repository` 명령어가 필요한데,  
         * ansible의 apt, apt_repository 모듈 내에는 해당 명령어가 없음
      5. 생각보다 중간에 누락되는 패키지 요소가 많아서 계속 `--fix-missing` 명령문을 중간에 넣어줘야 했다.
* python 관련 라이브러리 및 프로그램 설치와 관련된 오류들

![스크린샷, 2020-07-31 14-14-41](https://user-images.githubusercontent.com/58680504/89006772-6be9ad00-d342-11ea-9868-1a78fdcf4af1.png)

![스크린샷, 2020-07-31 15-15-35](https://user-images.githubusercontent.com/58680504/89006776-6d1ada00-d342-11ea-82e6-b24063e560b5.png)

![스크린샷, 2020-07-31 15-16-30](https://user-images.githubusercontent.com/58680504/89006779-6d1ada00-d342-11ea-9d16-58a02265ba36.png)

![스크린샷, 2020-07-31 15-17-01](https://user-images.githubusercontent.com/58680504/89006782-6db37080-d342-11ea-81b2-4a83bb5f9f1d.png)

![스크린샷, 2020-07-31 15-17-28](https://user-images.githubusercontent.com/58680504/89006784-6e4c0700-d342-11ea-94e2-03ac9c72d054.png)



* ansible-playbook 실행이 모두 제대로 완료된 모습이다.

![스크린샷, 2020-07-31 15-19-38](https://user-images.githubusercontent.com/58680504/89006785-6e4c0700-d342-11ea-852e-248345e7719c.png)



* 생성된 가상머신 접속

![스크린샷, 2020-07-31 15-19-52](https://user-images.githubusercontent.com/58680504/89006787-6ee49d80-d342-11ea-9dc7-e1c66e99c7f9.png)



* 설치된 파이썬 버전 및 파이썬3.6 확인

![스크린샷, 2020-07-31 15-20-02](https://user-images.githubusercontent.com/58680504/89006788-6f7d3400-d342-11ea-9897-bd96c675b886.png)

* ansible 설치 확인

![스크린샷, 2020-07-31 15-20-14](https://user-images.githubusercontent.com/58680504/89006789-7015ca80-d342-11ea-8570-b47b8a2dc289.png)



* 호스트( 로컬 )와 마운트한 폴더가 잘 작동하는지 확인

![스크린샷, 2020-07-31 15-20-23](https://user-images.githubusercontent.com/58680504/89006791-70ae6100-d342-11ea-96d9-2e44d9bddccc.png)



* playbook으로 설치한 기타 프로그램이 잘 설치되었는지 확인( git, unzip..) 

![스크린샷, 2020-07-31 15-20-34](https://user-images.githubusercontent.com/58680504/89006794-7146f780-d342-11ea-850a-4223c4e176e8.png)



* `Vagrantfile` 생성할 가상머신에 대한 설정파일

![스크린샷, 2020-07-31 15-23-49](https://user-images.githubusercontent.com/58680504/89006797-7146f780-d342-11ea-9b58-bd22cd89aabd.png)



* `ansible-playbook`을 실행하기 위한 `main.yml` 파일

![스크린샷, 2020-07-31 15-24-23](https://user-images.githubusercontent.com/58680504/89006798-72782480-d342-11ea-82cd-01f89730f079.png)





#### 3. Docker와 Ansible, Vagrant

* 추가적으로 Docker 파일을 ansible 파일 내에 만들어준다.

* `/ansible/docker/tasks` 경로로 `mkdir`

* 해당 경로에 main.yml 생성( `sudo` )

* `main.yml`파일에 다음과같이 작성한다.

  ![스크린샷, 2020-07-31 17-42-15](https://user-images.githubusercontent.com/58680504/89017474-516cff00-d355-11ea-89ee-4d9c4f57ad34.png)



* `./vagrant ssh`로 접속, 
* 가상 머신 내에서 ansible-playbook 명령어를 실행해준다.
* 다음과 같이 실행되는 모습을 볼 수 있다.

![스크린샷, 2020-07-31 15-55-46](https://user-images.githubusercontent.com/58680504/89017455-4d40e180-d355-11ea-8fc4-e452cce33c56.png)





* `docker --version`으로 잘 설치가 되었는지 확인한다.

![스크린샷, 2020-07-31 17-36-16](https://user-images.githubusercontent.com/58680504/89017457-4e720e80-d355-11ea-9140-3db509c47368.png)



* docker-compose도 잘 설치가 되었는지 확인을 했는데, 다음과 같은 오류가 뜬다.
* `ImportError: No module named zipp`

![스크린샷, 2020-07-31 17-36-23](https://user-images.githubusercontent.com/58680504/89017459-4f0aa500-d355-11ea-8f14-42ed5a7dd335.png)



* ` pip install zipp`을 시도, 모듈 설치

![스크린샷, 2020-07-31 17-37-03](https://user-images.githubusercontent.com/58680504/89017462-4f0aa500-d355-11ea-9c92-05f6cc8c32bd.png)



* 설치 실패 후, 업그레이드 하라고해서 추가적으로 업그레이드도 진행하였다.

![스크린샷, 2020-07-31 17-37-11](https://user-images.githubusercontent.com/58680504/89017464-4fa33b80-d355-11ea-9c43-50bbdc48a8a5.png)



* 업그레이드 후 설치가 잘 되었다.

![스크린샷, 2020-07-31 17-37-29](https://user-images.githubusercontent.com/58680504/89017467-503bd200-d355-11ea-8fb5-b0f373cbcb6b.png)



* 또 다른 모듈의 부재로 에러가 일어났다.

![스크린샷, 2020-07-31 17-37-54](https://user-images.githubusercontent.com/58680504/89017469-503bd200-d355-11ea-84c6-3f564261b0a2.png)



* 마찬가지로 pip로 설치를 시도하였다.

![스크린샷, 2020-07-31 17-38-08](https://user-images.githubusercontent.com/58680504/89017471-50d46880-d355-11ea-92f4-51fdcd138d34.png)



* 모든 모듈을 설치하고나니 docker-compose가 잘 설치된 것을 볼 수 있었다.

![스크린샷, 2020-07-31 17-38-16](https://user-images.githubusercontent.com/58680504/89018047-30f17480-d356-11ea-91f9-c9a1549dd084.png)

