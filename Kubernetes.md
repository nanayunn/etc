# Kubernetes



## 1. 쿠버네티스란?

* 컨테이너를 쉽고 빠르게 배포/확장하고 관리를 자동화해주는 오픈소스 플랫폼
* 단순한 컨테이너 플랫폼이 아닌 마이크로서비스, 클라우드 플랫폼을 지향
*  컨테이너로 이루어진 것들을 손쉽게 담고 관리할 수 있는 그릇 역할
* 서버리스, CI/CD, 머신러닝 등 다양한 기능이 쿠버네티스 플랫폼 위에서 동작



### 쿠버네티스의 특징

* 1. 거대한 커뮤니티와 생태계 보유

  <img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/cncf-map.png" alt="Cloud Native Landscape" style="zoom: 50%;" />



* 2. 다양한 배포 방식

  <img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/workload.png" alt="쿠버네티스 배포 방식" style="zoom:33%;" />

  * `Deployment`, `StatefulSets`, `DaemonSet`, `Job`, `CronJob`등 다양한 배포 방식
    * `Deployment` : 
      * 새로운 버전의 애플리케이션을 다양한 전략으로 무중단 배포
    * `StatefulSets` :
      * 순서나 데이터가 중요한 경우 사용
      * 실행순서를 보장
      * 호스트 이름과 볼륨을 일정하게 사용 가능
    * `DaemonSet` : 
      * 로그나 모니터링 등 노드에 설치가 필요한 경우
    * `Job`, `CronJob` : 
      * 배치성 작업



* 3. Ingress 기능

  * 다양한 웹 애플리케이션을 하나의 로드 밸런서로 서비스

    * 내부망 : 애플리케이션을 설치

    * 외부망 : `ALB`나 `Nginx`, `Apache`를 프록시 서버로 활용

      > 프록시 서버 : 도메인과 Path 조건에 따라 등록된 서버로 요청을 전달

    * => 외부에서 애플리케이션에 직접 접근할 수 없도록!

  * 하나의 클러스터에 여러 개의 Ingress 설정 가능

    * 관리자 접속용 Ingress / 일반 접속용 Ingress



* 4. 클라우드 지원

  * `Cloud Controller`을 이용하여 클라우드 연동을 손쉽게 확장할 수 있음
  * 다양한 클라우드 업체에서 모듈을 제공
    * AWS, 구글 클라우드, 마이크로소프트 애저 등등..



* 5. Namespace & Label

  * 하나의 클러스터를 논리적으로 구분하여 사용 가능

    * 기본 namespace : 

      * `system`, `default`
      * 이외 여러 개의 namespace 사용 가능

      <img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/namespace-label.png" alt="Namespace &amp; Label" style="zoom:33%;" />



* 6. RBAC( rold-based access control )

  * 접근권한 시스템
  * 유저별, 클러스터별, namespace별 CRUD 권한 적용 가능

  <img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/rbac.png" alt="Role based access control" style="zoom:33%;" />



* 7. CRD( Custom Resource Definitioin )

  * 쿠버네티스가 제공하지 않는 기능 확장 가능



* 8. Auto-Scaling

  * CPU, 메모리 사용량에 따른 확장 가능
  * HPA( Horizontal Pod AutoScaler )
    * 컨테이너 개수 조정
  * VPA( Vertical Pod AutoScaler )
    * 컨테이너의 리소스 할당량 조정
  * CA( Cluster AutoScaler )
    * 서버 개수 조정



* 9. Federation, Multi Cluster

  * 여러 개의 클러스터를 묶어서 관리할 수 있는 기능
    * 구글의 Anthos





### 쿠버네티스 핵심 = "상태"

<img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/desired-state.png" alt="Desired state" style="zoom: 33%;" />

* **desired state**

  * Current state을 모니터링하며 desired state을 유지하려는 것!

  * imperative(명령):

    * current state => desired state
    * 예 : 
      * nginx 컨테이너를 80 포트로 오픈.

  * declarative(선언):

    * desired state에 대한 정보

      * 80 포트를 오픈한 nginx 컨테이너를 1개 유지

      



### 쿠버네티스 객체

#### 1. Pod

<img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/pod.png" alt="Pod" style="zoom:50%;" />

* 쿠버네티스에서 배포할 수 있는 가장 작은 단위
  * 구성 : 
    * 한 개 이상의 컨테이너
    * 스토리지
    * 네트워크 속성
  * Pod에 속한 컨테이너는 스토리지와 네트워크 공유
    * 서로의 localhost도 접근 가능
  * 컨테이너가 하나더라도 무조건 Pod의 단위로 배포, 관리





#### 2. ReplicaSet

<img src="https://subicura.com/assets/article_images/2019-05-19-kubernetes-basic-1/replicaset.png" alt="ReplicaSet" style="zoom: 50%;" />

* Pod를 한 개 이상 복제하여 관리하는 객체
* Pod 생성 및 갯수 유지
* 구성
  * 복제할 갯수
  * Pod 갯수를 체크할 라벨 선택자
  * 생성할 Pod의 설정값(템플릿)
* 주로 Deployment와 같은 다른 오브젝트에 의해 사용



#### 3. Service

* 네트워크와 관련된 오브젝트
  * Pod를 외부 네트워크와 연결
  * 여러 개의 Pod를 대상으로하는 내부 로드 밸런서 생성
  * Service Discovery 
    * 내부 DNS에 서비스 이름을 도메인으로 등록



#### 4. Volume

* 저장소와 관련된 객체
* 대부분의 저장방식을 지원















