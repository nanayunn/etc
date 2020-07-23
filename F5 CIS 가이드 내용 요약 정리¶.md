# F5 CIS 가이드 내용 요약 정리[¶](https://redmine.piolink.com/issues/72416#F5-CIS-가이드-내용-요약-정리)



## 1. F5 Container Ingress Services 개요

**F5 CIS( F5 Container Ingress Services)**는.. 

* 컨테이너 오케스트레이션 환경과 통합하여,

  1. F5 BIG-IP 시스템에서 L4 / L7 서비스를 동적으로 생성하고,
  2. 서비스 전체에서 네트워크 트래픽을 로드 밸런싱합니다.

  

* 오케스트레이션 API 서버를 모니터링하여 
  
  * 컨테이너화 된 애플리케이션의 변경 사항에 따라 BIG-IP 시스템 구성을 수정할 수 있습니다.



|       CIS        |         오케스트레이션 환경          |
| :--------------: | :----------------------------------: |
| `k8s-bigip-ctlr` | `Kubenetes` 또는 `Red hat OpenShift` |

> `k8s-bigip-ctlr`
>
> * F5 BIG-IP 컨트롤러
> * Kubernetes 또는 OpenShift를 BIG-IP 오케스트레이션 플랫폼으로 사용할 수있는 클라우드 네이티브 커넥터



#### CIS 아키텍쳐 구성도

![Solution design: The Container Connector runs as an App within the cluster; it configures the BIG-IP device as needed to handle traffic for Apps in the cluster](https://clouddocs.f5.com/containers/v2/_images/cc_solution.png)

* 시나리오 흐름도

  1. 개발자가 어플리케이션을 수정하면 CIS는 변경 사항 발생을 감지
  2.  오케스트레이션 명령을 F5 SDK / iControl REST 호출로 변환
  3.  BIG-IP 시스템 구성을 동적으로 수정

  

  * CIS가 실행될 때
    * => 오케스트레이션 API를 관찰
    * => BIG-IP 시스템의 구성을 업데이트



#### CIS의 활용

* Active-Standby 상태의 BIG-IP 페어, 혹은 다수의 장치 클러스터를 관리하고 싶을 때 CIS를 활용할 수 있습니다.

  * 배포를 위한 세부 사항은 플랫폼에 따라 달라집니다.
  * 동일한 기본 원칙:
    * **각각의 BIG-IP 디바이스마다 하나의 BIG-IP Controller 인스턴스를 실행해야 합니다.**

  

* **예시**

  * 장치 : Active-Standby 상태의 BIG-IP 페어(2 개의 device)
  * 목적 : 단일 BIG-IP 파티션을 이용하여 Kubernetes 클러스터를 관리
  * **예시 아키텍쳐**
  * ![A diagram showing a BIG-IP active-standby device pair and 2 BIG-IP Controllers, running on separate nodes in a Kubernetes Cluster.](https://clouddocs.f5.com/containers/v2/_images/bigip-ha.png)
  * HA 설정 : 
    * 한 쌍의 BIG-IP 장치에 대해 2개의 BIG-IP 컨트롤러 인스턴스 배포
      * 장치 하나당 하나의 인스턴스
    * HA를 보다 명확히 하고 싶다면, 클러스터 내 분리된 Node에 각각 배포하면 됩니다. 



#### BIG-IP config sync ( 설정 동기화 )

* 각각의 컨테이너 커넥터는 설정 변경을 감지하기 위해 BIG-IP 파티션을 모니터링 합니다.
  * 변경 감지 시 각 커넥터는 자신의 설정을 BIG-IP에 **재적용**

* 다른 설정 유틸리니나, TMOS, 혹은 다른 장치나 서비스 그룹을 이용하여 설정을 동기화 하는 방안은 권장하지 않습니다. ( 오류 발생 및 서비스 중단 )



##### Kubernetes 및 OpenShift와 관련하여 주의사항

> * CIS는 FDB 항목과 ARP 레코드를 사용하여 BIG-IP 노드와 관련된 클러스터 리소스를 식별합니다.
>
> * BIG-IP config-sync에는 FDB 항목 또는 ARP 레코드가 포함되어 있지 않습니다.
>   * BIG-IP Controller로 BIG-IP HA 쌍 또는 클러스터를 관리할 때 자동 구성 동기화를 사용하지 않는 것을 권장.



* 설정 자동 동기화 시

  * 장애 발생 시 서비스 중단 창 팝업
    * 활성화 되려는 Standby device와 FDB, ARP 레코드를 업데이트 하려는 BIG-IP 컨트롤러 사이에서 발생
    * 중단 시간 디폴트 값 : 30초
      * 서비스 중단 시간을 단축시키려면 기본적인 원칙( 디바이스 하나 당 하나의 인스턴스 ) 만들 것.

  

#### BIG-IP SNAT 및 SNAT 오토 맵

* 컨테이너 커넥터로 작성된 모든 가상서버는 BIG-IP SNAT 오토 맵을 사용 (default)

  * 원본 IP 주소를 BIG-IP 시스템의 변환 주소 pool에 매핑을 가능하게 함.

  * BIG-IP 시스템이 클러스터 네트워크 내에서 연결을 처리할 때, IP 주소 풀에서 사용 가능한  변환 주소를 선택

  * SNAT 오토 맵은 고정 IP 보다 유동 IP를 선호

    * 한 쌍, 혹은 다수의 장치들 간의 failover를 지원

    

* **주의사항**
  * SNAT 오토맵이 현재 속한 VXLAN에서 사용가능한 유동 IP를 찾지 못한다면, 다른 VLAN의 변환 주소를 사용할 수도 있습니다.
    * 이런 경우에는 클러스터에 트래픽을 전송할 수 없습니다.





## 2. Kubernetes와 관련된 가이드



### 1. 릴리즈 및 버전 관리

* 컨테이너 통합 구성 요소

| Component                        | Version       |
| :------------------------------- | :------------ |
| BIG-IP Controller for Kubernetes | v1.0.x-1.13.x |



* CIS 호환성

  * CIS와 호환되는 버전은 아직 테스트 중에 있습니다.

  

* BIG-IP 컨트롤러/플랫폼 호환성

| Connector                                        | Version(s)  | Platform   | Version(s)  | BIG-IP version(s) | AS3         |
| :----------------------------------------------- | :---------- | :--------- | :---------- | :---------------- | :---------- |
| BIG-IP Controller for Kubernetes`k8s-bigip-ctlr` | v1.10-v1.14 | Kubernetes | v1.13-v1.16 | v12.x-v14.x       | v3.13-v3.17 |





### 2. 자주 묻는 질문 ( FAQ )

#### 2-1. CIS (Container Ingress Services)란?

*  주요 컨테이너 및 PaaS 환경에서 BIG-IP (Ingress Control) 앱 서비스를 지원하기 위해 제공되는 F5 오픈 소스 솔루션입니다.
*  DevOps의 컨테이너화 된 응용 프로그램 배포를위한 앱 셀프 서비스 선택 및 서비스 자동화가 가능합니다.
* Kubernetes 및 Red Hat OpenShift Platform as a Service (PaaS)와 같은 기본 컨테이너 환경 관리 / 오케스트레이션 시스템과 통합됩니다.

* CIS 구성요소
  - `bigip-controller-<environment>`: 
    - 특정 컨테이너 환경으로 BIG-IP 제어 평면을 통합
  - Kubernetes 용 F5 BIG-IP 컨트롤러
  - OpenShift 용 F5 BIG-IP 컨트롤러



#### 2-2. CIS의 최신 버전은 무엇입니까?

* BIG-IP Controller for Kubernetes v1.15 
* BIG-IP Controller for Red Hat OpenShift v3.x, v4.x

* 위의 두가지 버전이 최신 버전이며, Docker Hub, Github, Redhat에서 다운로드 가능합니다.



#### 2-3. 어떤 플랫폼이 검증 되었습니까?

* 다음의 플랫폼에서 검증되어 있습니다.
  * K8S v1.15
  * BIG-IP 14.X, BIG-IP 13.X, BIG-IP 12.X with OSCP
  * BIG-IP 14.X, BIG-IP 13.X, BIG-IP 12.X with K8S



#### 2-4. CIS 1.12에서 알아야 할 몇 가지 제한 사항은 무엇입니까?

* K8S 버전 1.13.4 또는 OSCP 버전 4.1 이상에서 노드 포트 모드로 작동할 때는 마스터 노드 레이블을 `node-role.kubernetes.io/master=true`로 설정해야 합니다. 
  * 설정하지 않으면 BIG-IP가 마스터 노드를 다른 pool 멤버로 취급합니다.

* 설정값에 관계 없이, `CIS는 secure-serverssl` 값을 `true`로 인식합니다.



#### 2-5. CIS와 관련하여 F5의 포지셔닝은 무엇입니까?

* CIS는
  * DevOps 오케스트레이션 내의 애플리케이션 서비스에서 셀프 서비스 선택을 활성화하여 애플리케이션 제공을 가능케 하고,
  * 이벤트 검색을 기반으로 스핀 업/다운을 자동화하는 **컨테이너 환경**을 위한 동적 애플리케이션 서비스 통합입니다.

  

  * 기존의 기본 앱 배포 워크 플로 및 새로운 URL 재 작성 경로와 통합하여 사용자의 경험과 생산성을 향상시킵니다.

  

  * BIG-IP 용 제어 평면 커넥터를 컨테이너 환경 관리 및 오케스트레이션 시스템에 통합합니다.
    * CIS는 Kubernetes 및 Red Hat OpenShift를 지원합니다.

  

  * BIG-IP는 컨테이너 수신 서비스를 통해 
  * HTTP 라우팅, URI 라우팅 및 컨테이너 환경으로의 API 버전 관리를 포함한 수신 제어 서비스를 활성화 할 수 있습니다. 
  * Ingress 서비스에는 로드 밸런싱, 스케일링, 보안 서비스 및 프로그래밍 기능이 포함됩니다.

  

  * 사전 구성된 Kubernetes Helm 차트를 사용하여 CIS 배포를 단순화하고 ,
  * 이전에 존재하던 설정을 기반으로 하는 유연한 배포를 제공합니다.



#### 2-6.  CIS에 대한 고객 가치 제안은 무엇입니까?

* CIS는 컨테이너 환경과의 native 통합을 통해 앱 성능 및 보안 서비스 오케스트레이션 및 관리를 위한 F5 인라인 통합을 제공합니다.

* 1. 

  * 컨테이너 애플리케이션의 오케스트레이션 매니지먼트 내에서 셀프 서비스를 선택하고,
  * 앱 이벤트를 기반으로 자동 검색 및 서비스를 삽입 할 수 있습니다. 

* 2.  

  * 컨테이너 앱 트래픽의 성능, 보안 및 관리를 위해 
  * 컨테이너 환경 내에서 수신 제어를 위한 인라인 BIG-IP의 적절한 서비스 구성을 쉽게 생성할 수 있습니다.

* 3.  

  * 미리 구성된 Kubernetes 리소스를 사용하여 
  * Helm Charts의 배포를 단순화하고 
  * 기존 구성으로 OpenShift Routes를 유연하게 배포할 수 있습니다.



#### 2-7. 대상 고객은 누구입니까?

* 네트워크 설계자,
* 네트워크 기업들,
* AppDev, 
* DevOps 및 시스템 인프라



* 셀프 서비스 및 표준화를 위한 컨테이너 관리 / 오케스트레이션 시스템을 통해 NetOps, Network Architects 고객, 애플리케이션 소유자 및 시스템 인프라 팀에  BIG-IP 서비스를 노출 할 수 있는 기능을 제공합니다.



* 여기서 NetOps의 경우,
  * DevOps 프로세스에 대한 AppDev/System 팀의 수많은 컨테이너 애플리케이션 서비스 IT 요청을 따라가지 못하는 경우가 많습니다.
  * 컨테이너 오케스트레이션 UI 내에서 셀프 서비스 통합 및 애플리케이션 서비스 자동화를 통해  이러한 문제를 해결할 수 있습니다.





#### 2-8. 컨테이너 수신 서비스 사용 사례는 무엇입니까?

- **컨테이너 환경을위한 동적 앱 서비스** : 

  - BIG-IP는 컨테이너 오케스트레이션 환경 내에서 애플리케이션 레벨 객체 (VIP, pool, pool 멤버)를 프로비저닝하고 관리 할 수 있습니다.

    -  앱 서비스 요구에 따라 pool 멤버를 자동으로 확장하거나 축소 할 수 있습니다.

    

- **클라우드 및 사내 컨테이너 환경에서의 자동 확장 및 보안** : 

  - 사내 및 클라우드 컨테이너 애플리케이션에 대한 이벤트 발생 감지를 기반으로 오케스트레이션 UI 또는 자동 앱 성능 및 보안 서비스 내에서 셀프 서비스 선택할 수 있습니다.

  

- **고급 컨테이너 앱 보호** :

  - BIG-IP 및 컨테이너 환경에 통합 된 컨테이너 수신 서비스는 간소화된 중앙 집중식 앱 및 네트워크 보호 기능을 제공합니다. 
  - 이 솔루션은 취약성 평가와 통합되어 F5의 공격 통찰력을 얻고 Prometeus, Splunk 또는 SIEM/Analytics 솔루션으로 데이터 스트림을 내보냅니다.

  

- **앱 마이그레이션을 능률화하고 여러 앱 버전을 동시에 확장** : 

  - Red Hat OpenShift PaaS에서 여러 앱 버전을위한 블루 / 그린 배포를 **동시에 수행**하여 새로운 애플리케이션으로 확장 및 이전 할 수 있습니다. 
  - Red Hat OpenShift에서 두 개 이상의 앱 버전에 대한 A / B 테스트 트래픽 관리 기능을 제공하여 동시에 개발 및 테스트 할 수 있습니다.



#### 2-9. 'Ingress'란 무엇이며 'ingress'와 어떻게 다른가요?

* `Ingress`는 HTTP 라우팅 또는 클러스터 서비스에 도달하기위한 규칙 모음을 나타냅니다. 
* `ingress`는 인바운드 연결, 앱로드 밸런싱 및 보안 서비스를 나타냅니다.







### 3. BIG-IP와 Flannel VXLAN 통합

#### 3-1. Kubernetes 환경에서 Flannel을 이용한 클러스터 네트워킹

> Flannel 
>
> * 3 계층 네트워크 패브릭
> * 컨테이너에 IP 를 연결하는 가상 네트워크



* Flannel의 역할

* 1. 클러스터 내의 각각의 노드에 Pod로써 실행

  * `flanneld`:
    * Flannel 데몬
    * 노드에게 네트워크 정보를 제공
    * Kubernetes API 서버의 노드에 대한 정보를 읽어옴
    * 모든 노드에서 실행
      * 클러스터 내의 모든 Pod는 서로간의 통신이 가능

* 2. 각 Kubernetes 노드에 서브넷 할당
     * 노드에서 실행되고 있는 각각의 Pod에 서브넷 내의 IP 주소를 할당



#### 3-2. BIG-IP 장치 및 Kubernetes 클러스터 네트워크

* BIG-IP 디바이스가 Kubernetes 클러스터 네트워크에 속해있을 때, 

  * 클러스터 내의 모든 Pod를 대상으로 바로 로드밸런싱을 할 수 있습니다.
    * Flannel과 BIG-IP 컨트롤러를 통해 BIG-IP가 각 포드의 `public-ip`주소를 찾을 수 있기 때문!

  

  * 작동 원리
    1. BIG-IP 디바이스가 VXLAN 터널을 통해 Flannel 네트워크에 접속합니다.
    2. BIG-IP 컨트롤러가 Flannel 네트워크에 대한 정보로 터널을 채웁니다. Flannel 네트워크에 대한 정보는 다음과 같습니다.
       * Kubernetes의 각 노드에 있는 Flannel VCLAN 인터페이스의 MAC 주소와 노드의 IP 주소를 맵핑하는 FDB 레코드
       * Flannel VXLAN 인터페이스의 MAC 주소와 Pod의 Flannel `public-ip`를 맵핑하는 고정 ARP 엔트리
    3. BIG-IP 컨트롤러는 각 Pod의 Flannel `public-ip`주소를 BIG-IP 노드에 할당합니다.

  

  * **예시**

    * | Kubernetes 노드 1                        |                   |
      | :--------------------------------------- | ----------------- |
      | 노드 IP 주소                             | 172.16.2.10       |
      | 노드의 flannel VXLAN interface MAC 주소  | 98:ba:76:dc:54:fe |
      | Flannel에 의해 할당된 Pod public-ip 주소 | 10.244.1.2        |

    * BIG-IP 컨트롤러는 BIG-IP 시스템의 노드에 FDB 레코드와 고정 ARP 엔트리를 만들기 위해 위와 같은 정보를 이용합니다.

    * FDB 레코드와 고정 ARP 엔트리 예시

      FDB record

      ```yaml
      //FDB 레코드
      flannel_vxlan {
       records [
          98:ba:76:dc:54:fe {
            endpoint: 172.16.2.10
          }
       ]
      }
      ```

      ```yaml
      //고정 ARP 엔트리
      {
         name: k8s-10.244.1.2
         ipaddress: 10.244.1.2
         macaddress: 98:ba:76:dc:54:fe
      }
      ```

    * 위와 같이 생성된 레코드 정보를 통해 BIG-IP 디바이스는 노드1의 Pod가 

    * “10.244.1.2”의 IP 주소를 가진 BIG-IP 노드로부터 트래픽을 전달받아야 한다는 사실을 알 수 있습니다.



* 기본적으로 BIG-IP 컨트롤러는 생성하는 모든 가상 서버에 BIG-IP SNAT 오토 맵을 사용합니다. 
* `k8s-bigip-ctlr`v1.5.0 부터는 SNAT 오토 맵을 사용하는 대신 Controller Deployment에서 특정 SNAT pool을 지정할 수 있습니다.
* BIG-IP가 클러스터 네트워크에 연결되는 환경에서 BIG-IP VTEP로 사용된 self IP는 클러스터 내의 모든 origin 주소에 대한 SNAT pool 역할을합니다.
*  self IP를 작성할 때 제공하는 서브넷 마스크는 SNAT pool에 사용 가능한 주소를 정의합니다.



#### 3-3. Flannel이 BIG-IP를 인식하는 방법

* Flannel의  `kube-subnet-manager` 는 Kubernetes 노드에 대한 정보를 얻기 위해 Kubernetes API를 이용합니다.
  * 따라서, BIG-IP 장치를 Flannel 네트워크에 추가하고 싶다면, **BIG-IP 장치를 Kubernetes의 노드로 추가해두어야 합니다**. 

* BIG-IP 장치를 Kubernetes의 새로운 노드로 추가하였다면,
  * 노드의 리소스에 Flannel 주석과 podCIDR를 추가합니다.
  * 노드를 올리고 실행시키면, Flannel이 추가된 주석을 인식한 뒤, VXLAN에 BIG-IP 디바이스를 추가할 것 입니다.





### 4. Flannel VXLAN에 BIG-IP 디바이스 추가를 위한 가이드

> Flannel VXLAN을 이용한 Kubernetes 클러스터에 BIG-IP 디바이스를 어떻게 추가하면 되는지에 대한 상세 가이드입니다.



* 가이드라인 순서

| Step | Task                                                         |
| :--- | :----------------------------------------------------------- |
| 1.   | Kubernetes에 대한 Flannel 배포                               |
| 2.   | BIG-IP 시스템 설정<br />1. VXLAN 터널 생성<br />2. VXLAN에 self IP 생성<br />3. VXLAN에 유동 self IP 생성<br />4. BIG-IP 객체 생성 확인 |
| 3.   | Flannel 오버레이 네트워크에 BIG-IP 장치 추가<br />1. VTEP MAC 주소 탐색<br />2. Flannel 주석 탐색<br />3. BIG-IP 디바이스에 대한 Kubernetes 노드 생성 |



* 다음은 가이드 라인 순서에 따른 가이드라인 별 설명입니다.

  **BIG-IP 시스템 설정**

* 1. VXLAN 터널 생성

  * TMOS 쉘에 로그인하세요.

    ```
    tmsh
    ```

  *  `flooding-type none`으로 VXLAN 프로파일을 만드세요.

    ```
    create net tunnels vxlan fl-vxlan port 8472 flooding-type none
    ```

  * VXLAN 터널을 생성하세요.

    * VXLAN 오버레이를 지원할 네트워크의 IP 주소를 `local-address` 로 설정하세요.
    * 모든 클러스터 리소스에 BIG-IP 장치에 대한 액세스 권한을 부여하기 위해서  `key`값을  `1`로 설정하세요.

    ```
    create net tunnels tunnel flannel_vxlan key 1 profile fl-vxlan local-address 172.16.1.28
    ```

  

  2. VXLAN에 self IP 생성

  * 1. BIG-IP 시스템에 할당할 Flannel 서브넷을 미리 정해두세요.

    * Kubernetes 클러스터의 노드에 이미 할당된 서브넷과 겹치지 않도록 주의하세요.
    * 이 서브넷은 BIG-IP 디바이스를 위한 '더미' 노드에 할당 됩니다.

  * 2. TMOS 쉘에 로그인하세요.

    ```
    tmsh
    ```

  * BIG-IP 디바이스에 할당하고 싶은 서브넷의 주소를 이용하여 self IP를 생성합니다.

    * self IP의 범위는 클러스터 서브넷 마스크 내에 있어야 합니다.
    * Flannel 네트워크의 default 서브넷 마스크는 `/16` 입니다.

    ```
    create net self 10.129.2.3 address 10.129.2.3/24 allow-service none vlan flannel_vxlan
    ```

  

  3. VXLAN에 유동 self IP 생성

  * 1. BIG-IP 디바이스에 할당한 Flannel 서브넷에 유동 self IP를 생성하세요.

    ```
    create net self 10.129.2.4 address 10.129.2.4/24 allow-service none traffic-group traffic-group-1 vlan flannel_vxlan
    ```

  

  4. 생성된 객체 확인

  * TMOS 쉘을 이용하여 생성된 객체를 확인할 수 있습니다.

    ```
    list net tunnels tunnel flannel_vxlan
    list net self 10.129.2.3
    list net self 10.129.2.4
    ```

  

  

  **Flannel 오버레이 네트워크에 BIG-IP 장치 추가**

  > Flannel은 클러스터 네트워크 내의 노드를 인식하기 위해 커스텀 주석을 이용합니다.

  1. VTEP MAC 주소 탐색

  * TMOS 쉘을 사용하여 BIG-IP VXLAN 터널의 MAC 주소를 찾을 수 있습니다.

    ```
    show net tunnels tunnel flannel_vxlan all-properties
    -------------------------------------------------
    Net::Tunnel: flannel_vxlan
    -------------------------------------------------
    MAC Address                   ab:12:cd:34:ef:56
    ...
    ```

  2. Flannel 주석 탐색

  * 클러스터 내의 모든 노드를 대상으로 **kubectl describe**를 실행하세요.

  * 노드 디스크립션에 Flannel 주석을 기록하세요.

    ```
    kubectl describe nodes
    ...
    flannel.alpha.coreos.com/backend-data:'{"VtepMAC":"<mac-address>"}'
    flannel.alpha.coreos.com/backend-type: 'vxlan'
    flannel.alpha.coreos.com/kube-subnet-manager: 'true'
    flannel.alpha.coreos.com/public-ip: <node-ip-address>
    ...
    ```

  3. BIG-IP 디바이스에 대한 Kubernetes 노드 생성

     1. "더미" Kubernetes 노드 리소스를 생성합니다.

        > 더미 노드는 항상 "NotReady" 상태에 놓여있는 노드입니다.
        >
        > 즉, Kubertes가 리소스를 스케줄링할 수 있는 노드가 아니며,
        >
        > BIG-IP 디바이스의 오버레이 네트워크 기능에 영향을 미치지 않습니다.

     * 모든 Flannel 주석에 대하여, BIG-IP VXLAN 데이터의 `backend-data` 와 `public-ip`주석을 정의하세요.

     ```
     flannel.alpha.coreos.com/backend-data:'{"VtepMAC":"<BIG-IP_mac-address>"}'
     flannel.alpha.coreos.com/public-ip: <BIG-IP_vtep-address>
     ```

     2. self IP와 유동 self IP를 작성할 때 사용했던 서브넷으로 `podCIDR`을 설정하세요.

     ```yaml
     apiVersion: v1
     kind: Node
     metadata:
       name: bigip
       annotations:
         # BIG-IP VXLAN tunnel의 MAC 주소를 입력하세요.
         flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"ab:12:cd:34:ef:56"}'
         flannel.alpha.coreos.com/backend-type: "vxlan"
         flannel.alpha.coreos.com/kube-subnet-manager: "true"
         # BIG-IP VTEP에 할당한 IP 주소를 입력하세요.
         flannel.alpha.coreos.com/public-ip: 172.16.1.28
     spec:
       # BIG-IP device에 할당하고 싶은 flannel subnet을 정의하세요.
       # 겹치치 않도록 주의하세요.
       podCIDR: 10.129.3.0/24
     ```

     3. Kubernetes API 서버에 노드 리소스를 업로드 하세요.

     ```
     kubectl create -f f5-kctlr-bigip-node.yaml
     ```

     4. 노드 생성을 확인하세요.

     ```
     kubectl get nodes
     NAME           STATUS    AGE       VERSION
     bigip          NotReady  5m        v1.7.5
     k8s-master-0   Ready     2d        v1.7.5
     k8s-worker-0   Ready     2d        v1.7.5
     k8s-worker-1   Ready     2d        v1.7.5
     ```

     

### 5. Kubernetes 환경에서 BIG-IP 컨트롤러 설치

> Kubernetes에서 BIG-IP 컨트롤러를 설치할 때 Deployment 기능을 이용하십시오.
>
> 이 가이드문은 표준 Kubernetes 환경에 기반한 것입니다.



* 가이드라인 순서

| Step | Task                                                         |
| :--- | :----------------------------------------------------------- |
| 1.   | 초기 셋팅                                                    |
| 2.   | RBAC 인증 설정                                               |
| 3.   | Deployment 생성<br />1. 기본 Deployment 설정<br />2. Health Check 실행<br />3. BIG-IP SNAT pool과 오토맵 사용<br />4. Flannel BIG-IP 통합을 위한 Deployment |
| 4.   | Kubernetes API 서버에 리소스 업로드                          |
| 5.   | Pod 객체 생성 확인                                           |



**초기 셋팅**

1. BIG-IP HA( High Availability )를 이용하고 싶다면, DSC( Device Service Cluster )에 2개 이상의 F5 BIG-IP를 설정하세요.
2. BIG-IP 시스템에 새로운 파티션을 설정합니다.
   * BIG-IP 컨트롤러는 `/Common` 파티션에 속한 객체를 제어할 수 없습니다.
   * [옵션] 
     * Controller는 BIG-IP에서 구성한 IP 주소를 Route Domain 식별자로 선언할 수 있습니다 . 
     * 다수의 어플리케이션에 동일한 IP 주소 공간을 사용하여 별도의 파티셔닝이 필요한 경우, Route Domain을 이용할 수 있습니다. 
       * 1) BIG-IP 시스템에서 파티션을 생성 한 후 Route Domain을 생성하고
       * 2) 라우트 도메인을 파티션의 기본값으로 할당합니다. 
     * [옵션] BIG-IP HA 쌍 또는 클러스터를 사용하는 경우 그룹간의 변경 사항을 동기화하십시오.
3. BIG-IP 로그인 자격 증명을 Secret에 저장하세요.
4. 개인 Docker 레지스트리에서 `k8s-bigip-ctlr`이미지를 가져와야 하는 경우, Docker 로그인 자격 증명을 Secret 형태로 저장하세요. 



**RBAC 인증 설정**

1. BIG-IP 컨트롤러 서비스 계정을 생성하세요.

   ```
   kubectl create serviceaccount bigip-ctlr -n kube-system
   serviceaccount "bigip-ctlr" created
   ```

2. 클러스터 Role과 클러스터 Role Bining을 생성하세요.

   * 아래의 permission set은 필요에 따라 변경하여 사용합니다.

   ```yaml
   # k82 클러스터에서만 사용할 수 있습니다.
   
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: bigip-ctlr-clusterrole
   rules:
   - apiGroups: ["", "extensions"]
     resources: ["nodes", "services", "endpoints", "namespaces", "ingresses", "pods"]
     verbs: ["get", "list", "watch"]
   - apiGroups: ["", "extensions"]
     resources: ["configmaps", "events", "ingresses/status"]
     verbs: ["get", "list", "watch", "update", "create", "patch"]
   - apiGroups: ["", "extensions"]
     resources: ["secrets"]
     resourceNames: ["<secret-containing-bigip-login>"]
     verbs: ["get", "list", "watch"]
   
   ---
   
   kind: ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: bigip-ctlr-clusterrole-binding
     namespace: <controller_namespace>
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: bigip-ctlr-clusterrole
   subjects:
   - apiGroup: ""
     kind: ServiceAccount
     name: bigip-ctlr
     namespace: <controller_namespace>
   
   ```

   

**Deployment 생성**

* 유효한 YAML이나 JSON 형식을 이용하여 Kubernetes Deployment를 정의하세요.
* `replica` 항목값을 증가시키지 마십시오.  복제 컨트롤러 인스턴스의 실행은 오류를 야기할 수 있습니다.
* BIG-IP 컨트롤러의 모든 기능을 실행하려면 관리자 권한이 필요합니다.



1. 기본 Deployment 형식

   * 아래의 예제는 Kubernetes에서 BIG-IP 컨트롤러를 실행시키기 위해 필요한 기본 config 파라미터 값들의 예제입니다.

   ```yaml
   apiVersion: extensions/v1beta1
   kind: Deployment
   metadata:
     name: k8s-bigip-ctlr-deployment
     namespace: kube-system
   spec:
     # DO NOT INCREASE REPLICA COUNT
     replicas: 1
     template:
       metadata:
         name: k8s-bigip-ctlr
         labels:
           app: k8s-bigip-ctlr
       spec:
         # Name of the Service Account bound to a Cluster Role with the required
         # permissions
         serviceAccountName: bigip-ctlr
         containers:
           - name: k8s-bigip-ctlr
             image: "f5networks/k8s-bigip-ctlr"
             env:
               - name: BIGIP_USERNAME
                 valueFrom:
                   secretKeyRef:
                     # Replace with the name of the Secret containing your login
                     # credentials
                     name: bigip-login
                     key: username
               - name: BIGIP_PASSWORD
                 valueFrom:
                   secretKeyRef:
                     # Replace with the name of the Secret containing your login
                     # credentials
                     name: bigip-login
                     key: password
             command: ["/app/bin/k8s-bigip-ctlr"]
             args: [
               # See the k8s-bigip-ctlr documentation for information about
               # all config options
               # https://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/latest
               "--bigip-username=$(BIGIP_USERNAME)",
               "--bigip-password=$(BIGIP_PASSWORD)",
               "--bigip-url=<ip_address-or-hostname>",
               "--bigip-partition=<name_of_partition>",
               "--pool-member-type=nodeport",
               "--agent=as3",
               ]
         imagePullSecrets:
           # Secret that gives access to a private docker registry
           - name: f5-docker-images
           # Secret containing the BIG-IP system login credentials
           - name: bigip-login
   
   ```

   

2. Health Check 실행

   * Kubernetes에는 두 가지의 health check 타입이 있습니다.

   1. Readiness Probes : 
      * Pod가 언제 ready 상태가 되는지 확인
      * 컨테이너가 트래픽을 수용할 수 있는 시기를 결정
      * 서비스의 백엔드로 사용할 포드를 제어
      * 모든 컨테이너의 상태가 'ready'가 되면 Pod도 'ready' 상태라고 간주
      * Pod가 준비되지 않은 경우 로드 밸런서 서비스에서 제거

   2. Liveness Probes : 

      * ready 상태가 된 Pod가 healthy한지 unhealthy한지 판단
      * 컨테이너를 언제 restart할지 판단
        * 컨테이너 restart는 어플리케이션의 가용성을 향상
      * 컨테이너가 응답하지 않으면 멀티 스레딩 결함으로 인해 응용 프로그램이 교착 상태가 될 수 있음

      

   * 컨테이너 상태를 체크하기 위해서 이용할 수 있는 방법

     * Pod에 대한 HTTP 요청
     * Pod에 대한 명령 실행
     * Pod에 대한 TCP 요청
     * d

   * HTTP 메소드를 이용하는 Deployment의 예:

     | 파라미터            | 설명                                                         |
     | :------------------ | :----------------------------------------------------------- |
     | periodSeconds       | kubelet이 liveness probe 3초마다 수행하도록 지정             |
     | initialDelaySeconds | 첫번째 Probe를 수행하기 전에 kubelet에게 3초 대기를 알림     |
     | timeOutSeconds      | Probe가 실행을 마칠 때 까지의 대기 시간.<br />지정해둔 시간이 초과되면, OpenShift 컨테이너 플랫폼의 경우, Probe가 실패하였다고 간주. |

   * kubelet은 컨테이너의 Health check에 web hook을 이용합니다.

   * HTTP 응답 코드가 `200`~`399` 사이에 있으면 체크에 성공한 것 입니다.

   * Probe를 수행하기 위해, kubelet은 컨테이너에서 실행 중인 서버( 8080 port )에 HTTP GET 요청을 보냅니다.

     * 서버의 / health 경로에 대한 핸들러는 성공 코드를 리턴합니다.

   * 예 : 

     ```yaml
     livenessProbe:
        failureThreshold: 3
        httpGet:
           path: /health
           port: 8080
           scheme: HTTP
        initialDelaySeconds: 15
        periodSeconds: 15
        successThreshold: 1
        timeoutSeconds: 15
     readinessProbe:
        failureThreshold: 3
        httpGet:
           path: /health
           port: 8080
           scheme: HTTP
        initialDelaySeconds: 30
        periodSeconds: 30
        successThreshold: 1
        timeoutSeconds: 15
     ```

     * deployed pod의 health check를 원한다면, 다음 명령을 입력하면 됩니다.

       ```
       Kubectl describe pod <pod_name> -n kube-system
       ```

       * 결과

         ```yaml
         resources: {}
                 terminationMessagePath: /dev/termination-log
                 terminationMessagePolicy: File
                 - --log-level=debug
                 terminationMessagePolicy: Fileermination-log
               --log-level=debug
               --namespace=default
               --route-label=systest
               --insecure=true
               --agent=cccl
            Liveness:      http-get http://:8080/health delay=15s timeout=15s period=15s #success=1 #failure=3
            Readiness:     http-get http://:8080/health delay=30s timeout=15s period=30s #success=1 #failure=3
            Environment:   <none>
            Mounts:        <none>
         Volumes:          <none>
         ```

     * `curl http://<self-ip>:<port no>/health` 명령어 수행시, `OK` 응답을 받을 수 있습니다.





* 만약, BIG-IP 디바이스를 Flannel VXLAN을 통한 클러스터 네트워크에 연결하고 싶다면, Deployment 설정 시 다음 명령어를 포함하세요.
  * `--pool-member-type=cluster` 
  * `--flannel-name=/Common/tunnel_name`



3.  BIG-IP SNAT pool과 오토맵 사용

* 특정 SNAT 풀을 사용하려면 `k8s-bigip-ctlr` Deployment의 `args`섹션에 다음을 추가하세요 .	

  * `"--vs-snat-pool-name=<snat-pool>"`

  * `<snat-pool>`의 위치에 BIG-IP 디바이스의  `/Common` 파티션에 이미 존재하는 SNAT pool의 이름을 입력하세요.
  * *컨트롤러는 새로운 SNAT pool을 지정해주지 않습니다.*



**Kubernetes API 서버에 리소스 업로드**

* `kubectl apply`를 사용하여 Deployment, 클러스터 Role 및 클러스터 Role Binding을 Kubernetes API 서버에 업로드하세요.

  ```
  kubectl apply -f f5-k8s-bigip-ctlr_basic.yaml -f f5-k8s-sample-rbac.yaml [-n kube-system]
  deployment "k8s-bigip-ctlr" created
  cluster role "bigip-ctlr-clusterrole" created
  cluster role binding "bigip-ctlr-clusterrole-binding" created
  ```

  

**Pod 객체 생성 확인**

* `k8s-bigip-ctlr` Pod가 성공적으로 실행되었는지 확인하기 위해 다음 명령어를 실행하세요.

  * `kubectl get`

  ```
  kubectl get pods -n kube-system
  NAME                                  READY     STATUS    RESTARTS   AGE
  k8s-bigip-ctlr-331478340-ke0h9        1/1       Running   0          1h
  ```

  









