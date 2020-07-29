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



