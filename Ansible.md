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

* Inventory : 

  > ‘어디에서’ 앤서블을 실행하는가

  * 관리되는 노드의 목록
  * 호스트 파일이라고도 한다. 
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









