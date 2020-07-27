# F5 CIS 가이드 내용 요약 정리



## 1. F5 Container Ingress Services 개요

**F5 CIS( Container Ingress Services )**는.. 

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
      * 서비스 중단 시간을 단축시키려면 기본적인 원칙( 디바이스 하나 당 하나의 인스턴스 )을 적용해야합니다.

  

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
       * 2) Route Domain을 파티션의 기본값으로 할당합니다. 
   * [옵션] 
     * BIG-IP HA 쌍 또는 클러스터를 사용하는 경우 그룹간의 변경 사항을 동기화하십시오.
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

  



### 6. BIG-IP 객체 관리

> Kubernetes에서 서비스 및 수신을 위한 BIG-IP 객체를 배포할 수 있습니다.
>
> BIG-IP 컨트롤러는 아래 표에 명시된대로 BIG-IP 객체를 생성, 업데이트, 제거 및 관리할 수 있습니다.

![표](https://user-images.githubusercontent.com/58680504/88471651-9a344a80-cf46-11ea-939d-131478f82d42.png)

> SSL 프로파일에 대한 BIG-IP 컨트롤러 지원은 리소스 유형에 따라 다릅니다.
>
> - F5 리소스 : 기존 클라이언트 SSL profile 사용
> - Kubernetes Ingress : 기존 클라이언트 SSL profile 사용
> - OpenShift Routes : 새 클라이언트 또는 서버 SSL profile을 생성, 또는 기존 클라이언트 또는 서버 SSL profile 사용
>
> **모든** profile에 대한 BIG-IP 컨트롤러 지원은 기본 profile 형식에만 적용됩니다. 
>
> 최적화 및 사용자 정의된 버전은 지원하지 않습니다.



**객체 네이밍 규칙**

* BIG-IP 컨트롤러는 모든 BIG-IP 가상 서버 객체에 대해 다음과 같이 명명합니다.
  *  `[namespace]_[resource-name]`
  * 예 : 
    * namespace :  `default` 
    * ConfigMap name : `k8s.vs`  일때, 
    * object preface :  `default_k8s.vs_173.16.2.2:80`



**HA( High-availability )와 Multi-tenancy**

* 한 쌍의, 혹은 다수의 BIG-IP 디바이스를 사용하고 있다면,  **하나의 BIG-IP 디바이스 당 하나의 BIG-IP 컨트롤러**를 배포할 것을 권장합니다.





### 7. 서비스에 Virtual Server 연결

> F5 리소스를 사용하여 사용자 지정 BIG-IP 가상 서버를 Kubernetes 및 OpenShift의 서비스에 연결할 수 있습니다.



* F5 리소스 ConfigMap

  >  각각의 서비스를 외부 트래픽에 노출시켜줍니다.

  * 특징 
    * Ingresses 및 Routes 보다 뛰어난 유연성과 커스터마이징 옵션
    * L4 수신 (TCP 또는 UDP).
    * 비표준 포트에서 L7 수신. ( 예를 들면 8080 또는 8443 포트에서 )



**서버 연결 순서**

| 단계 | 작업                           |
| :--- | :----------------------------- |
| 1.   | 서비스를 위해 가상 서버 정의   |
| 2.   | ConfigMap을 API 서버에 업로드  |
| 3.   | BIG-IP 시스템의 변경 사항 확인 |



**서비스를 위해 가상 서버 정의**

* 생성하고 싶은 가상 서버를 F5 리소스의 JSON blob에 정의하세요.
* 정의 후, Kubernetes ConfigMap 리소스의 **data** 섹션에 해당 JSON blob를 포함시킵니다.



* HTTP 예시

  * 서비스의 형태가 다음과 같다면, 

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: myService
    labels:
      app: myApp
  spec:
    ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
    type: clusterIP
  ```

  * HTTP ConfigMap의 형식은 다음과 같을 것 입니다.

  ```yaml
  kind: ConfigMap
  apiVersion: v1
  metadata:
    name: myApp.vs
    labels:
      f5type: virtual-server
  data:
    # https://clouddocs.f5.com/containers/latest/releases_and_versioning.html#f5-schema
    schema: "f5schemadb://bigip-virtual-server_v0.1.7.json"
    data: |
      {
        "virtualServer": {
          "backend": {
            "servicePort": 80,
            }
          },
          "frontend": {
            "virtualAddress": {
              "port": 8080,
              "bindAddr": "1.2.3.4"
            },
            "partition": "k8s",
            "balance": "least-connections-member",
            "mode": "http"
          }
        }
  ```

* F5 리소스 옵션

  * ConfigMap의 servicePort 옵션은 Service의 port 옵션과 맵핑됩니다.
  * BIG-IP 컨트롤러는 Pod 노드의 포트와 BIG-IP 가상 서버의 엔드 포인트를 연결할 때 이 옵션을 사용합니다.
  * Service의 targetPort의 옵션값은 사용자가 트래픽을 전송하고 싶은 Pod나 컨테이너의 포트에 해당합니다.
  * 사용자는 balance: round-robin의 옵션값을 지원되는 BIG-IP 로드 밸런싱 모드 중 하나로 변경할 수 있습니다.



**ConfigMap을 API 서버에 업로드**

* 하나의 동일한 서버에 HTTP와 HTTPS 가상 서버를 생성하고 싶다면, 
  * 각각의 포트에 ConfigMap를 생성하면 됩니다.
  * 1. `apply` 옵션에서 YAML 파일 두 개를 이용하여 name을 전달하거나, 
  * 2. 단일 manifest 파일에 두 리소스에 대한 내용을 모두 포함시킬 수 있습니다.



* Kubernetest 서비스를 이용할 경우

  * namespace의 default값에 해당하지 않는 리소스를 업로드하고자 할 때,

    *  `--namespace`(또는 `-n`) 플래그를 사용하여 올바른 namespace를 지정하십시오 .

    ```
    kubectl apply -f <filename.yaml> [--namespace=<resource-namespace>]
    ```

* OpenShift 서비스를 이용할 경우

  * default 값에 해당하지 않거나, 현재 진행중인 프로젝트 내에 없는 리소스를 업로드하고자 할 때,

    *  `--namespace`(또는 `-n`) 플래그를 사용하여 올바른 프로젝트를 지정하십시오.

    ```
    oc apply -f <filename.yaml> [--namespace=<resource-project>]
    ```



**BIG-IP 시스템의 변경 사항 확인**

> BIG-IP 객체의 생성, 변경, 삭제에 대한 검증을 위해
>
> * BIG-IP configuration 유틸리티를 이용하거나,
> * TMOS 쉘을 이용할 수 있습니다.



* Configuration 유틸리티를 이용 시:

  * Local Traffic의 Virtual Servers 메뉴에 가서,
  * Partition 드롭다운 메뉴에서 올바른 파티션을 선택하면 됩니다.

  

* TMOS 쉘 이용 시 : ( 콘솔 이용 예시 )

```
admin@(bigip)(cfg-sync Standalone)(Active)(/Common) cd my-partition
admin@(bigip)(cfg-sync Standalone)(Active)(/my-partition) tmsh
admin@(bigip)(cfg-sync Standalone)(Active)(/my-partition)(tmos)$ show ltm virtual
------------------------------------------------------------------
Ltm::Virtual Server: default_myApp.vs_173.16.2.2_80
------------------------------------------------------------------
Status
  Availability     : available
  State            : enabled
  Reason           : The virtual server is available
  CMP              : enabled
  CMP Mode         : all-cpus
  Destination      : 173.16.2.2:80
...
Ltm::Virtual Server: default_myApp.vs_173.16.2.2_443
------------------------------------------------------------------
Status
  Availability     : available
  State            : enabled
  Reason           : The virtual server is available
  CMP              : enabled
  CMP Mode         : all-cpus
  Destination      : 173.16.2.2:443
...
```



* 서비스를 제거할 때
  * API 서버의 ConfigMap과 관련된 서비스를 제거하고자 할 때, BIG-IP 컨트롤러가 서비스와 관련된 BIG-IP 객체를 제거할 것 입니다.
  * 이후, 서비스와 관련된 F5 리소스 ConfigMap을 제거하여 주십시오.
* 서비스를 대체할 때
  * 새로운 서비스의 요구조건에 맞는 새로운 F5 리소스 ConfigMap을 생성하여야 합니다.



### 8. BIG-IP 컨트롤러를 수신 컨트롤러로 사용하기 

> * Kubernetes 용 BIG-IP 컨트롤러를 Kubernetes에서 Ingress 컨트롤러로 사용할 수 있습니다 . 
>
> * BIG-IP 컨트롤러의 Ingress 주석은 BIG-IP 시스템에서 필요한 트래픽 관리 객체를 정의합니다.



* helm을 이용한다면,  `f5-bigip-ingress chart`를 이용하여 리소스를 생성 및 관리할 수 있습니다.
  * BIG-IP 자체의 리소스를 생성 및 관리하고자 한다면, F5 Helm 차트를 이용하면 됩니다.



**컨트롤러 설정 가이드**

| 단계 | 기술                                        |
| :--- | :------------------------------------------ |
| 1.   | 가상 서버에 대한 BIG-IP self IP 주소를 생성 |
| 2.   | Ingress에 리소스를 사용하여 주석을 추가     |
| 3.   | Health 모니터링                             |
| 4.   | API 서버에 Ingress 업로드                   |
| 5.   | BIG-IP 시스템에서 객체 생성 여부 확인       |



**초기 설정 셋팅( 가상 서버에 대한 BIG-IP self IP 주소를 생성 )**

>  컨트롤러에 default IP 주소를 할당했거나, host IP 주소를 DNS 조회를 이용하여 확인하는 경우 이 단계를 건너뛰십시오.

* BIG-IP 시스템의 외부 네트워크에서 자체 IP 주소 를 할당하세요.
* 할당한 후, 해당 IP 주소를 Ingress 리소스에 할당합니다. 
  * 클러스터 모드 에서 BIG-IP 컨트롤러를 실행하는 경우, IP 주소는 **BIG-IP VXLAN 터널에 할당된 서브넷 내에** 있어야합니다 .



* unattached  pool을 만들려면 `Pools without virtual servers`옵션 상태의 pool을 참조하십시오 .



**주석 추가**

1. **`kubectl`을 이용**
   * 기존의 Ingress에 지원되는 Ingress 주석을 추가하고 싶다면, `kunectl 주석`을 사용하세요.
   * 모든 키-값 쌍을 단일 **kubectl 주석** 명령에 포함시키면 
     * BIG-IP 시스템의 부분 업데이트를 피할 수 있습니다.

* 다음 설정값을 바탕으로 한 예시를 참고하세요.

  * Ingress class : f5
    * 다른 컨트롤러와의 충돌을 피하기 위함
  * default IP : `k8s-bigip-ctlr Deployment`내의 주소
  * Listening port : 443
  * 파티션 : "k8s"
  * [옵션] : 라운드 로빈 로드 밸런싱
  * [옵션] : BIG-IP pool health monitor
  * [옵션] : HTTP 요청을 HTTPS로 리디렉션
  * [옵션] : HTTP 요청 거부

  

  * kubectl 주석 작성 예제

  ```
  kubectl annotate ingress myingress kubernetes.io/ingress.class="f5" \
                                     virtual-server.f5.com/ip="controller-default" \
                                     virtual-server.f5.com/http-port="443" \
                                     virtual-server.f5.com/partition="k8s" \
                                     virtual-server.f5.com/balance="round-robin" \
                                     virtual-server.f5.com/health='[{"path": "svc1.example.com/app1", "send": "HTTP GET /health/svc1", "interval": 5, "timeout": 10}]' \
                                     ingress.kubernetes.io/ssl-redirect="true" \
                                     ingress.kubernetes.io/allow-http="false"
  ```

  

2. `Ingress 리소스`를 이용

   * Ingress 주석 작성 예제

   ```yaml
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: myingress
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /
       kubernetes.io/ingresslass: "f5"
       virtual-server.f5.com/ip: "controller-default"
       virtual-server.f5.com/http-port: "443"
       virtual-server.f5.com/partition: "k8s"
       virtual-server.f5.com/balance: "round-robin"
       virtual-server.f5.com/health: '[{"path": "svc1.example.com/app1", "send": "HTTP GET /health/svc1", "interval": 5, "timeout": 10}]'
       ingress.kubernetes.io/ssl-redirect: "true"
       ingress.kubernetes.io/allow-http: "false"
   spec:
     rules:
     - http:
         paths:
         - path: /testpath
           backend:
             serviceName: test
             servicePort: 80
   ```

   



**Health monitor 세팅**

>  Kubernetes Ingress 리소스를 위해 가상 서버에 health monitor을 추가, 혹은 업데이트를 원한다면  `virtual-server.f5.com/health` 주석을 사용하세요.

* Health monitor 주석 작성 예제

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ing1
  namespace: default
  annotations:
    virtual-server.f5.com/ip: "controller-default"
    virtual-server.f5.com/partition: "k8s"
    virtual-server.f5.com/health: |
      [
        {
          "path":     "svc1.example.com/app1",
          "send":     "HTTP GET /health/app1",
          "interval": 5,
          "timeout":  10
        }, {
          "path":     "svc2.example.com/app2",
          "send":     "HTTP GET /health/app2",
          "interval": 5,
          "timeout":  5
        }
      ]
spec:
  rules:
  - host: svc1.example.com
    http:
      paths:
      - backend:
          serviceName: svc1
          servicePort: 8080
        path: /app1
  - host: svc2.example.com
    http:
      paths:
      - backend:
          serviceName: svc2
          servicePort: 9090
        path: /app2
```



**BIG-IP SSL profile 또는 Secret 사용**

> BIG-IP의 SSL profile이나, Kubernetes에서 제공하는 Secrets를 이용하여 수신 보안 설정을 할 수 있습니다.

1. `spec.tls`수신 리소스 섹션에 대한 인증서와 키를 포함하는 SSL 프로파일 또는 비밀을 지정하세요 .

2. `ingress.kubernetes.io/ssl-redirect`주석을 추가하세요. ( **OPTIONAL** ; 기본값은 `"true"`)

3. `ingress.kubernetes.io/allow-http`주석을 추가하세요. ( **OPTIONAL** ; 기본값은 `"false"`)

   > - Ingress 리소스에서 하나 이상의 SSL profile을 지정할 수 있습니다.
   > - TLS 수신 속성을 제공하지 않고 `spec.tls`섹션을 지정하면,
   >   -  BIG-IP 장치는 로컬 traffic policies를 사용하여 HTTP 요청을 HTTPS로 리디렉션합니다.



* 아래의 표에서는  `ingress.kubernetes.io/ssl-redirect`과 `ingress.kubernetes.io/allow-http`설정값에 따른 다양한 조합의 컨트롤러 작동 방식을 보여줍니다.

* | State | sslRedirect | allowHttp | Description                 |
  | :---- | :---------- | :-------- | :-------------------------- |
  | 1     | F           | F         | Just HTTPS, nothing on HTTP |
  | 2     | T           | F         | HTTP redirects to HTTPS     |
  | 2     | T           | T         | Honor sslRedirect == true   |
  | 3     | F           | T         | Both HTTP and HTTPS         |



* BIG-IP SSL profile을 이용한 TLS Ingress 예제

  ```yaml
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: ingressTLS
    namespace: default
    annotations:
      virtual-server.f5.com/ip: "1.2.3.4"
      virtual-server.f5.com/partition: "k8s"
      ingress.kubernetes.io/ssl-redirect: "true"
      ingress.kubernetes.io/allow-http: "false"
  spec:
    tls:
      # Provide the name of the BIG-IP SSL profile you want to use.
      - secretName: /Common/clientssl
    backend:
      # Provide the name of a single Kubernetes Service you want to expose to external
      # traffic using TLS
      serviceName: myService
      servicePort: 443
  ```

* Kubernetes Secret을 이용한 TLS Ingress 예제

  ```yaml
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: ingressTLS
    namespace: default
    annotations:
      virtual-server.f5.com/ip: "1.2.3.4"
      virtual-server.f5.com/partition: "k8s"
      ingress.kubernetes.io/ssl-redirect: "true"
      ingress.kubernetes.io/allow-http: "false"
  spec:
    tls:
      # Provide the name of the Secret you want to use.
      - secretName: myTLSSecret
    backend:
      # Provide the name of a single Kubernetes Service you want to expose to external
      # traffic using TLS
      serviceName: myService
      servicePort: 443
  
  
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: no-rules-map
  spec:
    tls:
    - secretName: testsecret
    backend:
      serviceName: s1
      servicePort: 80
  ```

  



**URLS 재작성**

* 라우팅을 목적으로 하였을 때, BIG-IP 컨트롤러는 URLs 설정값을 재작성할 수 있습니다.



**API 서버에 Ingress 업로드**

* 새롭게 생성되었거나 수정된 Ingress 리소스를 Kubernetes API 서버에 올리기 위해 `kubectl apply` 명령어를 이용하세요.

  > namespace의 default값에 해당하지 않는 리소스를 업로드하고자 할 때,
  >
  > *  `--namespace`(또는 `-n`) 플래그를 사용하여 올바른 namespace를 지정하십시오 .

```
kubectl apply -f <filename.yaml> [--namespace=<resource-namespace>]
```





**BIG-IP 시스템에서 생성된 객체 확인**

> 생성된 객체에 대하여 BIG-IP configuration이나 TMOS 쉘을 이용하여 객체의 생성, 변경 사항, 삭제 여부를 알 수 있습니다.

* TMOS 관리 콘솔 테스트 시 : 

  ```
  admin@(bigip)(cfg-sync Standalone)(Active)(/Common) cd my-partition
  admin@(bigip)(cfg-sync Standalone)(Active)(/my-partition) tmsh
  admin@(bigip)(cfg-sync Standalone)(Active)(/my-partition)(tmos)$ show ltm virtual
  ------------------------------------------------------------------
  Ltm::Virtual Server: default_myApp.vs_173.16.2.2_80
  ------------------------------------------------------------------
  Status
    Availability     : available
    State            : enabled
    Reason           : The virtual server is available
    CMP              : enabled
    CMP Mode         : all-cpus
    Destination      : 173.16.2.2:80
  ...
  Ltm::Virtual Server: default_myApp.vs_173.16.2.2_443
  ------------------------------------------------------------------
  Status
    Availability     : available
    State            : enabled
    Reason           : The virtual server is available
    CMP              : enabled
    CMP Mode         : all-cpus
    Destination      : 173.16.2.2:443
  ...
  ```

  

**가상 서버를 삭제하고자 할 때**

1. Ingress 정의 부분에서 BIG-IP 컨트롤러 주석을 삭제

2. Kubernetes API 서버 업데이트

   ```
   kubectl apply -f <filename.yaml> [--namespace=<resource-namespace>]
   ```

   



**Ingress 리소스 예제**



1. 단일 서비스 대상

   * *단일 서비스* Ingress는 단일 Kubernetes 서비스를 위한 BIG-IP 가상 서버 및 서버 풀을 만듭니다.

   ```yaml
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: singleIngress1
     namespace: default
     annotations:
       virtual-server.f5.com/ip: "1.2.3.4"
       virtual-server.f5.com/partition: "k8s"
   spec:
     backend:
       # The name of the Service you want to expose to external traffic
       serviceName: myService
       servicePort: 80
   ```



2. Simple Fanout

   * Simple Fanout Ingress는 다수의 Kubernetes 서비스를 위한 BIG-IP 가상 서버 및 서버 풀을 만듭니다.(하나의 서비스당 하나의 pool을 생성)

   ```yaml
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
     name: ing-fanout
     namespace: default
     annotations:
       virtual-server.f5.com/ip: "1.2.3.4"
       virtual-server.f5.com/partition: "k8s"
       virtual-server.f5.com/balance: "least-connections-node"
   spec:
     rules:
     - host: mysite.example.com
       http:
         paths:
         - path: /app1
           backend:
             serviceName: myService1
             servicePort: 80
         - path: /app2
           backend:
             serviceName: myService2
             servicePort: 80
   ```

   

3. 이름 기반 가상 호스팅

   * 이름 기반 가상 호스팅 Ingress는 다음과 같은 BIG-IP 객체들을 생성합니다.
     * 하나의 가상 서버
     * 하나의 서비스당 하나의 pool
     * 호스트 이름 및 경로를 기반으로 요청을 특정 pool로 라우팅하기위한 로컬 traffic policy.

   > `backend`Ingress 리소스 섹션에 이름이 지정된 서비스의 호스트 또는 경로를 지정하지 않으면
   >
   >  BIG-IP 장치는 모든 호스트 / 경로에 대한 트래픽을 프록시합니다.

   ```yaml
   # 호스트 지정 시
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
    name: ing-virtual-hosting
    namespace: default
    annotations:
     virtual-server.f5.com/ip: "1.2.3.4"
     virtual-server.f5.com/partition: "k8s"
     virtual-server.f5.com/balance: "least-connections-node"
     virtual-server.f5.com/http-port: "80"
   spec:
    rules:
    # URL
    - host: site1.example.com
      http:
        # path to Service from URL
        paths:
          - path: /app1
            backend:
              serviceName: myService1
              servicePort: 80
    # URL
    - host: site2.example.com
      http:
        # path to Service from URL
        paths:
          - path: /app2
            backend:
              serviceName: myService2
              servicePort: 80
   ```

   ```yaml
   # 모든 호스트를 대상으로 할 때
   apiVersion: extensions/v1beta1
   kind: Ingress
   metadata:
    name: ing-virtual-hosting
    namespace: default
    annotations:
     virtual-server.f5.com/ip: "1.2.3.4"
     virtual-server.f5.com/partition: "k8s"
     virtual-server.f5.com/balance: "least-connections-node"
     virtual-server.f5.com/http-port: "80"
   spec:
    rules:
    # Omit the host name (URL) to match all hosts
    - http:
        # Provide the path to each Service you want to proxy
        paths:
        - path: /app1
          backend:
            serviceName: myService1
            servicePort: 80
        - path: /app2
          backend:
            serviceName: myService2
            servicePort: 80
   ```

   



### 9. CIS와 AS3 확장 통합

> CIS (Container Ingress Services) 와 AS3 (Application Services 3) 을 확장하여
>
>  BIG-IP 오케스트레이션 플랫폼으로 사용할 수 있습니다.



* 전제 조건
  * BIG-IP 시스템 소프트웨어 버전 12.1.x 이상을 실행 중입니다.
  * BIG-IP 시스템에 AS3 확장 버전 3.10 이상이 설치되어 있습니다.
  * 관리자 권한을 가진 BIG-IP 시스템 사용자 계정.

* 제한 사항
  * AS3 pool 클래스 선언은 단일 로드 밸런싱 pool만 지원합니다.
  * CIS는 단일 AS3 ConfigMap 인스턴스만 지원합니다.
  * AS3은 BIG-IP 노드를 새 파티션으로 이동하는 것을 지원하지 않습니다.



* 선언형 API

  * AS3 Extensions는 선언형 API를 사용합니다.
    * AS3 Extension 선언은 BIG-IP 시스템의 원하는 구성 상태를 나타냅니다. 
  * AS3 Extensions을 사용할 때 CIS는 단일 Rest API 호출을 사용하여 선언 파일을 보냅니다.

  > BIG-IP 오케스트레이션에 AS3을 사용하려면 Deployment의 argument 섹션에 
  >
  > `–agent = as3` 옵션을 추가하세요.



* CIS 서비스 검색

  * CIS는 Service Discovery를 사용하여 BIG-IP 시스템의 로드 밸런싱 pool 멤버를 동적으로 발견 및 업데이트 할 수 있습니다. 
  * CIS는 레이블을 사용하여 AS3 템플릿에 속한 각 pool의 정의를 Kubernetes Service 리소스에 매핑합니다. 
  * 이 매핑을 만들려면 Kubernetes 서비스에 다음 레이블을 추가하십시오.

  | Label                           | 설명                                                         |
  | :------------------------------ | :----------------------------------------------------------- |
  | app: <string>                   | 이 label은 service와 deployment을 연결합니다.<br />*Important: 레이블 포함 필수, DNS에서 확인 되어야 함.* |
  | cis.f5.com/as3-tenant: <string> | AS3 선언 시 파티션의 이름<br />*Important: AS3 17 이상을 이용하는 경우 문자열에  hypen(-) 포함 가능.* |
  | cis.f5.com/as3-app: <string>    | AS3 선언 시 class 이름                                       |
  | cis.f5.com/as3-pool: <string>   | AS3 선언 시 pool 이름                                        |

> CIS K8s 컨트롤러는 Ingress Routes를 AS3 선언으로 변환하고 기존 AS3 선언을 동적으로 감지합니다. 
>
> 이후, 두 AS3 선언을 결합하고 통합 선언을 생성한 뒤, 특정 필드( 예를 들어, `action`필드 )를 통합 선언 안에서 삭제합니다. 

> **주의**
>
> 동일한 레이블 집합으로 태그가 지정된 여러 Kubernetes 서비스 리소스는 *CIS 오류 및 서비스 검색 실패를 유발합니다.*



**서비스 라벨 개요**

![../_images/k8s_service_labels.png](https://clouddocs.f5.com/containers/v2/_images/k8s_service_labels.png)

**deployment 예**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: f5-hello-world
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: f5-hello-world
  template:
    metadata:
      labels:
        app: f5-hello-world
    spec:
      containers:
      - env:
        - name: service_name
          value: f5-hello-world
        image: f5devcentral/f5-hello-world:latest
        imagePullPolicy: Always
        name: f5-hello-world
        ports:
        - containerPort: 80
          protocol: TCP
```

**서비스 예**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: f5-hello-world
  namespace: kube-system
  labels:
    app: f5-hello-world
    cis.f5.com/as3-tenant: AS3
    cis.f5.com/as3-app: f5-hello-world
    cis.f5.com/as3-pool: web_pool
spec:
  ports:
  - name: f5-hello-world
    port: 80
    protocol: TCP
    targetPort: 80
  type: NodePort
  selector:
    app: f5-hello-world
```



* AS3 오케스트레이션 활성화

  * 다음 단계를 거쳐 BIG-IP 오케스트레이션에서 AS3을 사용할 수 있습니다.

    1. Deployment의 argument 섹션에 `–agent=as3`를 포함합니다.

       > 이 예에서 `k8s-bigip-ctlr`은 **myParition_AS3** 파티션을 생성하여 pool 및 가상 서버와 같은 LTM 객체를 저장합니다. 
       >
       > FDB 레코드와 고정 ARP 항목은 **myPartition에** 저장됩니다 . 
       >
       > 이 파티션은 수동으로 관리하면 안됩니다.

       ```
       args: [
             "--bigip-username=$(BIGIP_USERNAME)",
             "--bigip-password=$(BIGIP_PASSWORD)",
             "--bigip-url=10.10.10.10",
             "--bigip-partition=myPartition",
             "--pool-member-type=cluster",
             "--agent=as3"
             ]
       ```

       

    2. 컨트롤러 재 시작

       ```
       kubectl apply -f f5-k8s-bigip-ctlr.yaml
       ```

       

* 서비스 검색 및 컨트롤러 모드

  * AS3 선언내의 IP 주소 및 서비스 포트 정보
  * CIS 서비스 검색은 컨트롤러 모드에 따라 다르게 추가됩니다.

  | 컨트롤러 모드 | Configuration 업데이트                                       |
  | :------------ | :----------------------------------------------------------- |
  | Cluster IP    | - Kubernetes `Service endpoint IP Addresses`를 `ServiceAddresses`섹션에 추가하세요.<br /> -`ServicePort`섹션의 엔트리를 변경할 때, Kubernetes `Service endpoint service ports`를 사용하십시오 . |
  | Node port     | Kubernetes `cluster node IP addresses`를 `ServerAddresses`섹션에 추가하세요.<br /> `ServicePort`섹션의 엔트리를 변경할 때, Kubernetes `cluster NodePort ports`를 사용하세요.<br />Kubernetes 서비스를 `Nodeport`유형으로 공개하세요 . |





* AS3 선언 처리

  > CIS를 사용하여 AS3 선언을 처리하기 위해, 
  >
  > `virtual-server`설정값은 `f5type`로, `as3`의 설정값은 `true`로 설정해주세요.

  * CIS는 AS3 데이터를 검증하기 위해 `gojsonschema`를 이용합니다.
    * 검증에 실패할 경우 에러 로그 발생
    * AS3의 라벨 값은 boolean 타입의 `True`가 아니라, string 타입의 `true`값 지정이 필요합니다.
  * AS3 ConfigMap의 예제

  ```yaml
  apiVersion: v1
  metadata:
    name: as3-template
    namespace: kube-system
    labels:
      f5type: virtual-server
      as3: "true"
  data:
    template: |
      {
            <YOUR AS3 DECLARATION>
      }
  ```

  * AS3 선언 처리에는 다음 네 단계가 포함됩니다.

  1. configMap 내에 AS3 템플릿을 제출하고 Kubernetes에 배포하십시오.
  2. AS3 configMap이 준비가 되면, CIS는 서비스 검색 기능을 수행합니다.
  3. 서비스 검색이 완료된 후, CIS는 AS3 템플릿을 수정하고 발견된 엔드 포인트를 추가합니다. CIS는 AS3 템플리트에서 다음 두 값만 수정합니다.
     - `serverAddresses` 배열. 
       - 이 배열이 비어 있지 않으면 CIS는 엔트리를 덮어 쓰지 않습니다.
     - `servicePort` 값.
  4. CIS는 생성된 AS3 선언을 BIG-IP 시스템에 업로드 한 뒤, 트래픽 처리를 시작합니다.

  

* CIS와 AS3 deployment 워크 플로우

![../_images/container_ingress_services.png](https://clouddocs.f5.com/containers/v2/_images/container_ingress_services.png)

* 파라미터 값

  | Parameter      | Type    | Required | Default | Description                                                  | Allowed Values  |
  | :------------- | :------ | :------- | :------ | :----------------------------------------------------------- | :-------------- |
  | as3-validation | Boolean | Optional | True    | AS3 유효성 검사를 수행할지 여부를 CIS에 알려줍니다.          | “true”, “false” |
  | insecure       | Boolean | Optional | False   | 유효하지 않은 SSL 인증서를 사용하여 BIG-IP와의 통신을 허용할지 여부를 CIS에 알려줍니다. | “true”, “false” |





* CIS Configmap 삭제
  * 선언형 API를 이용하여 작성된 configmap은 삭제하더라도 BIG-IP 시스템 configuration 설정값이 자동 삭제되지 않습니다.
  * 삭제를 원한다면, 새롭게 빈 configmap을 생성 및 배포한 후, 컨트롤러를 재 시작하세요.



* SSL 인증서 유효성

  * CIS는 기본 Debian/Redhat image와 함께 번들로 제공되는 root CA 인증서를 사용.

    * 따라서, CIS는 BIG-IP 시스템의 자체 서명 SSL 인증서의 유효성은 검증 불가.
    * 검증 오류 시 AS3 로그 파일에 다음과 유사한 오류 메시지를 기록합니다.

    ```
    [ERROR] [as3_log] REST call error: Post https://10.10.10.100/mgmt/shared/appsvcs/declare: x509: cannot validate certificate for 10.10.10.100
    ```

  * 이 문제를 피하기 위해

    - Kubernetes 배포를 실행할 때 구성에 `--insecure=true`옵션을 포함
      - 인증서 유효성 검사를 무시.
    - CIS가 신뢰할 수있는 인증서 저장소를 업데이트
      - BIG-IP 시스템과의 신뢰 옵션 설정 .



* 관리자 파티션
  * ARP 엔트리를 이용하여 서비스 검색을 하기 위해
    * CIS는 관리자 파티션을 필요로 함
  * Kubernetes deployment를 실행할 때
    * `--bigip-partition=<name>` 파라미터 값을 유니크한 값으로 설정 필요
      * 유니크하게 설정된 BIG-IP 파티션은 AS3 Tenant 클래스를 허용하지 않습니다.



* AS3 tenant

  > AS3 tenant란?
  >
  > * 특정 AS3 응용 프로그램을 지원하는 구성을 그룹화하는 데 사용되는 BIG-IP 관리 파티션
  *  AS3 어플리케이션은 네트워크 기반 비즈니스 응용 프로그램 또는 시스템을 지원.
  *  AS3 tenant는 다른 tenant의 애플리케이션이 공유하는 자원을 포함할 수 있습니다.



### 10. BIG-IP 가상 서버 관리

> BIG-IP 컨트롤러를 사용하여 BIG-IP 가상 서버 및 pool을 Kubernetes 및 OpenShift 환경의 서비스에 연결할 수 있습니다.



* 기존 가상 서버 편집

  > 서비스, 수신 또는 라우팅과 관련된 BIG-IP 가상 서버를 수정

  - 리소스 YAML 또는 JSON 파일을 원하는대로 변경하십시오.

  - 다음을 사용하여 Kubernetes 또는 OpenShift API 서버에 파일을 업로드하십시오.

  - Kubernetes

    ```
    kubectl apply -f <filename.yaml> [--namespace=<resource-namespace>]
    ```

  - OpenShift

    ```
    oc apply -f <filename.yaml> [--namespace=<resource-project>]
    ```

    

* Ingress 와 Routes

  * **annotate** 명령을 사용하여 Ingress 또는 Route 리소스에 대한 BIG-IP Controller Annotations를 추가 / 변경

  * 예 : 로드 밸런싱 모드를 `least-connections-member`로 변경

    * Kubernetes Ingress

    ```
    kubectl annotate ingress myIngress virtual-server.f5.com/balance=least-connections-member [--namespace=**myNamespace**]
    ```

    * OpenShift Route

    ```
    oc annotate route myRoute virtual-server.f5.com/balance=least-connections-member [--namespace=**myProject**]
    ```

    

* Health monitor 추가 및 삭제

  > 서비스 사용 가능 여부를 확인하는 데 사용되는 BIG-IP 상태 모니터를 추가하거나 수정

  1. 원하는 health monitor를 정의 / 편집.
  2. 상태 모니터를 F5 리소스 ConfigMap 의 `backend`섹션에 추가
  3. API 서버 업데이트

  * 예제  YAML

  ```yaml
  kind: ConfigMap
  apiVersion: v1
  metadata:
    name: myApp.vs
    labels:
      f5type: virtual-server
  data:
    # See the f5-schema table for schema-controller compatibility
    # https://clouddocs.f5.com/containers/latest/releases_and_versioning.html#f5-schema
    schema: "f5schemadb://bigip-virtual-server_v0.1.7.json"
    data: |
      {
        "virtualServer": {
          "backend": {
            "servicePort": 80,
            "serviceName": "myService",
            "healthMonitors": [{
              "interval": 30,
              "protocol": "http",
              "send": "GET HTTP/1.1/\r\n",
              "recv": "200|OK",
              "timeout": 120
            }]
          },
          "frontend": {
            "virtualAddress": {
              "port": 8080,
              "bindAddr": "1.2.3.4"
            },
            "partition": "k8s",
            "balance": "least-connections-member",
            "mode": "http"
          }
        }
      }
  ```

  

* 가상 서버 삭제

  * Kubernetes 또는 OpenShift 리소스를 삭제하면 BIG-IP 컨트롤러는 관련된 모든 BIG-IP 객체를 삭제

  * 예시

    * API 서버에서 "myService"라는 서비스를 삭제

    ```
    kubectl delete service myService [--namespace = ** <service_namespace> **] \ kubernetes
    oc delete service * myService [--namespace = ** <service_project> **] \ openshift
    ```

    * BIG-IP 컨트롤러는 BIG-IP 디바이스에서 해당 가상 서버, pool등을 제거
    * 서비스 삭제 시
      * 서비스와 관련된 F5 Resource ConfigMap, Ingress 및 / 또는 Route도 삭제 필요
      * 관련 설정을 삭제하지 않고 새로운 서비스를 동일한 이름으로 생성하게 되면,
        * 삭제가 안된 설정값을 기반으로 서비스가 재생성
    * 설정들이 삭제되었는지 확인( TMOS 쉘 이용 )

    ```
    admin@(bigip)(cfg-sync Standalone)(Active)(/Common) cd my-partition
    admin@(bigip)(cfg-sync Standalone)(Active)(/my-partition) tmsh
    admin@(bigip)(cfg-sync Standalone)(Active)(/my-partition)(tmos)$ show ltm virtual
    admin@(bigip)(cfg-sync Standalone)(Active)(/my-partition)(tmos)$
    ```

    

* 서비스 다운시키고자 할 때
  * 유지 관리를 위해 서비스를 중단해야 하는데 BIG-IP에서 서비스 개체를 잃고 싶지 않은 경우:
    *  먼저 BIG-IP 컨트롤러를 제거.
      * 컨트롤러는 종료될 때 어떤 개체도 삭제하지 않으므로 BIG-IP 시스템 구성에 영향을 주지 않고 제거할 수 있습니다. 
    * 이후 컨트롤러를 백업하면 오케스트레이션 시스템의 현재 상태와 일치하도록 BIG-IP 구성이 업데이트



* 가상 서버가 없는 pool

  > BIG-IP 컨트롤러를 사용하여 frontend 가상 서버에 *연결되지 않은* BIG-IP pool ( unattached pool ) 생성이 가능
  >
  > * unattached pool 이용시 : 
  >   * IPAM 시스템을 이용하여 가상서버에 IP 주소 할당 필요
  >   * IPAM 시스템을 이용하지 않을 경우, BIG-IP 시스템에 traffic을 pool로 라우팅하는 다른 방법이 있어야 함 ( 예 :  iRules 또는 로컬 traffic policy )

  

  * `unattached pool` 생성 방법

    1. `bindAddr` 옵션 없이 F5 리소스 생성

       * 생성 예제

       ```yaml
       kind: ConfigMap
       apiVersion: v1
       metadata:
         # name of the resource to create on the BIG-IP
         name: http.pool_only
         labels:
           f5type: virtual-server
       data:
         schema: "f5schemadb://bigip-virtual-server_v0.1.7.json"
         data: |
           {
             "virtualServer": {
               "backend": {
                 "servicePort": 80,
                 "serviceName": "myService",
                 "healthMonitors": [{
                   "interval": 30,
                   "protocol": "http",
                   "send": "GET HTTP/1.1/\r\n",
                   "recv": "200|OK",
                   "timeout": 120
                 }]
               },
               "frontend": {
                 "virtualAddress": {
                   "port": 8080
                 },
                 "partition": "k8s",
                 "balance": "ratio-member",
                 "mode": "http"
               }
             }
           }
       ```

    2. Kubernetes API 서버 업데이트

    3.  IPAM을 이용하여 가상서버에 pool을 연결,

       * 혹은 BIG-IP configuration 유틸리티 / TMSH 를 이용하여 올바른 라우팅 규칙을 이용, pool 멤버를 iRule, traffic policy에 추가



* IPAM을 이용하여 가성 서버에 pool 연결

  * 할당한 IP 주소로 F5 리소스 ConfigMap에 주석을 달 수 있도록 IPAM 시스템 설정. 아래의 주석을 이용

    `virtual-server.f5.com/ip=<ip_address>`

  * BIG-IP 컨트롤러가 주석을 발견하면,

    * 지정된 IP 주소로 새로운 BIG-IP 가상 서버를 생성하고
    * 기존 pool을 가상 서버에 연결

  > 컨트롤러는 기존 BIG-IP 가상 서버에 unattached pool에 연결 하는 것을 지원하지 않습니다.
  >
  > -  이미 연결된 pool이 있는 기존 가상 서버에 pool을 추가하려고하면 오류가 표시.
  > - `/ Common` 파티션에 있는 기존 가상 서버를 사용하려고하면 ,
  >   - 컨트롤러가 관리 파티션 내에서 새로운 가상 서버를 만들려고 시도하므로 충돌 오류가 표시.
  > - 관리 파티션에서 가상 서버를 수동으로 생성하면 컨트롤러가 가상 서버를 삭제합니다.





* 가상 서버에서 pool 분리

  > 가상 서버를 삭제하더라도 pool / pool member는 유지

  1. 가상 서버의 F5 리소스 ConfigMap에서 `bindAddr`필드를 제거
  2. API 서버를 업데이트
  3. **kubectl get** (Kubernetes) 또는 **oc get** (OpenShift)을 사용하여 변경 사항을 확인





### 11. F5 리소스 설명

> F5 리소스는 BIG-IP 객체 집합을 정의하는 특수 형식의 JSON Blob
>
> F5 리소스를 사용하여 가상 서버를 만들면 BIG-IP Controller가 BIG-IP 시스템에서 생성되는 객체를 제어.



* F5 리소스 ConfigMap을 사용하여 애플리케이션에 대한 사용자 정의 가상 서버를 작성할 수 있습니다.
  * 예 : 
  * 비표준 포트를 사용하는 HTTP 가상 서버
  * BIG-IP Controller Ingress 주석 이상의 사용자 정의가 필요한 HTTP 가상 서버 
  * 비 HTTP 수신이 필요한 TCP 또는 UDP 응용 프로그램 용 가상 서버



**F5 리소스 특성**

| Property | Description                                                  | Required |
| :------- | :----------------------------------------------------------- | :------- |
| data     | JSON 객체                                                    | Required |
| frontend | BIG-IP 객체를 정의                                           |          |
| backend  | - 프록시하려는 서비스를 식별.<br />- 서비스에 대한 BIG-IP 상태 모니터를 정의 |          |
| f5type   | BIG-IP 컨트롤러가 감시하는 `label`의 특성.                   | Required |
| schema   | 인코딩 된 데이터를 해석하는 방법을 BIG-IP 컨트롤러에 알려줍니다. | Required |



**Data. Frontend**

* 특정 서비스에 대해 BIG-IP 시스템에서 작성하려는 객체를 정의
* `virtualServer` 특성을 이용하여 사용자 지정 가상 서버를 정의할 수 있습니다.



**Data.Backend**

* 백엔드 서버 풀을 구성하는 Kubernetes 서비스를 식별 하기 위해 사용
  * 이 섹션에서 BIG-IP health monitor도 정의 가능



**F5type**

> F5type?
>
> * Kubernetes 자원 메타 데이터 레이블
> * 작업 툴(Kubernetes, OpenShift)에 따라 사용법이 다릅니다.

* Kubernetes
  * F5 Resource ConfigMap에서 `f5type` 설정값 : 
    *  `virtual-server` 
  * BIG-IP 컨트롤러에게 BIG-IP 디바이스에 가상 서버를 생성하고 싶음을 알려줍니다.
* OpenShift
  * Route Resource에서  `f5type` 설정값 : 
    * `--route-label`설정
    * BIG-IP 컨트롤러가 API 서버를 감시할 경로를 지정



**F5 schema**

> F5 schema?
>
> * 구조를 정의하고 F5 리소스에 정의된 데이터를 포맷팅.
> * BIG-IP 컨트롤러는 제공된 schema 버전을 사용하여 다른 입력을 확인

* 모든 BIG-IP Controller 버전은 이전 버전과 호환되지만 이전 schema를 사용하면 Controller 기능이 제한될 수 있습니다. 
* 전체 기능 세트에 액세스하려면 컨트롤러 버전에 해당하는 최신 schema 버전을 사용해야합니다.







### 12. BIG-IP 컨트롤러 모드

* **pool-member-type** 옵션
  *  컨트롤러가 노드 포트 또는 클러스터 내에서 실행되는 모드를 결정

1. Node port 모드

   * Kubernetes에서 BIG-IP 컨트롤러에 대한 기본 작동 모드

   * 모든 Kubernetes Cluster Networks를 지원

   * 특정 BIG-IP 라이센스 요구 사항이 없기 때문에 설정하기가 쉬움

   * 2계층 로드 밸런싱 사용

     * 1. BIG-IP 플랫폼이 요청을 노드 (kube-proxy)로 로드 밸런싱
     * 2. Pod에 대한 노드 (kube-proxy)로드 밸런싱 요청.
     * ![../_images/k8s_nodeport.png](https://clouddocs.f5.com/containers/v2/_images/k8s_nodeport.png)

     

   * Node port 모드 사용 시 제한사항 : 

     * Kubernetes 서비스도 **NodePort 유형을** 사용해야 함
     * BIG-IP 시스템은 Pod에 직접 로드 밸런싱을 수행 할 수 없음
       * L7 지속성( L7 persistence )과 같은 일부 BIG-IP 서비스가 작동하지 않을 수도 있음.
       * 추가 네트워크 대기 시간 필요.
       * Pod health에 대해 BIG-IP 컨트롤러가 갖고 있는 제한적인 가시성



2. Cluster 모드

   * BIG-IP 장치를 Kubernetes 클러스터 네트워크에 통합 하려면 클러스터 모드를 사용 .

   * SDN 서비스 및 고급 라우팅이 포함된 Better 또는 Best 라이센스가 필요

   * 노드 포트 모드에 비해 클러스터 모드가 가지는 뚜렷한 이점 : 

     - 모든 유형의 Kubernetes Services를 사용할 수 있습니다.
     - BIG-IP 시스템은 클러스터의 모든 Pod에 직접 로드 밸런싱하여 다음을 제공:
       - L7 지속성을 포함한 BIG-IP 서비스
       -  Kubernetes API를 통해 BIG-IP 컨트롤러가 Pod health를 완전히 파악.

     ![../_images/k8s_cluster.png](https://clouddocs.f5.com/containers/v2/_images/k8s_cluster.png)

   * **네트워크 추가 구성 필요**

     * BIG-IP 컨트롤러가 클러스터 네트워크에 대해 수행해야하는 작업을 고려해야 합니다.
       * Pod에 대한 BIG-IP 시스템 구성은 **항상** 자동으로 관리.
       * 노드 및 클러스터에 대한 작업의 경우, 일부 수동으로 관리
     * BIG-IP 컨트롤러가 수행하는 일반적인 작업들 : 
       * 기존 서비스에서 Pod를 추가 또는 제거하거나 Pod가 있는 서비스를 노출.
       * 클러스터에서 노드를 추가하거나 제거.
       * 새로운 Kubernetes Cluster를 맨 첫 단계부터 생성.





### 13. BIG-IP 컨트롤러를 Kubernetes Ingress 컨트롤러로 사용

> BIG-IP 컨트롤러는 다음과 같은 Kubernetes Ingress 리소스 타입을 지원합니다.
>
> * 단일 서비스
> * Simple Fanout
> * 이름 기반 가상 호스팅
> * TLS



* 다중 Ingress 컨트롤러 사용하기

  > BIG-IP 컨트롤러는 `ingress.class`로 정의되지 않은 모든 Ingress 리소스를 자동으로 관리합니다 .

  * `ingress.class` 속성은 기본적으로 비어있음
  * `ingress.class` 속성값이 "f5"가 아닌 모든 값은 무시
  * 다른 Ingress 컨트롤러를 사용하여 Kubernetes Ingress 리소스를 관리하는 경우 :
    1. BIG-IP 컨트롤러가 관리하는 모든 Ingress 리소스에서 `ingress.class` 값을 "f5"로 설정하세요.
       * 예 : `kubernetes.io/ingress.class="f5"`
    2. 다른 Ingress 컨트롤러가 관리하는 Ingress 리소스에서 적절한 것을 `ingress.class`로 정의하세요 .



* IP 주소 할당

  * Controller는 Ingress 리소스에 나열된 각 고유 IP 주소에 대해 하나의 가상 서버를 생성합니다.

  * 아래 옵션을 사용하여 IP 주소 할당을 관리

    1. BIG-IP SNAT pool 및 SNAT 오토 맵 사용

    2. 기본 공유 IP 주소 설정

       * BIG-IP Controller 인스턴스 당 `default-ingress-ip`는 하나만 정할 수 있습니다 .

    3. 사용 DNS 조회

       * BIG-IP 컨트롤러는 기본적으로 DNS 조회를 사용하여 호스트 이름을 확인

         > Ingress 리소스 `spec.rules.host`섹션에 제공된 첫 번째 호스트 이름

         * 호스트의 IP 주소가 확인되면, Ingress의 가상 서버에 할당합니다.

    4. IPAM 시스템 이용





### 14. Kubernetes에서 BIG-IP 오케스트레이션에 AS3 사용 

> `k8s-bigip-ctlr`은 BIG-IP 오케스트레이션을 위한 AS3( Application Service 3 )을 사용할 수 있습니다.
>
> * 선언형 API를 사용하여 고유한 파티션 내에서 객체 생성



* 전제 조건
  * 버전 v12.1.x 이상의 BIG-IP 시스템.
  * AS3 v3.11 이상의 BIG-IP 시스템.
  * **관리자** 역할을 가진 BIG-IP 사용자 계정

1. AS3 오케스트레이션 활성화

* 다음 단계를 거쳐 BIG-IP 오케스트레이션에서 AS3을 사용할 수 있습니다.

  1-1. Deployment의 argument 섹션에 `–agent=as3`를 포함합니다.

  > 이 예에서 `k8s-bigip-ctlr`은 **myParition_AS3** 파티션을 생성하여 pool 및 가상 서버와 같은 LTM 객체를 저장합니다. 
  >
  > FDB 레코드와 고정 ARP 항목은 **myPartition에** 저장됩니다 . 
  >
  > 이 파티션은 수동으로 관리하면 안됩니다.

  ```
  args: [
        "--bigip-username=$(BIGIP_USERNAME)",
        "--bigip-password=$(BIGIP_PASSWORD)",
        "--bigip-url=10.10.10.10",
        "--bigip-partition=myPartition",
        "--pool-member-type=cluster",
        "--agent=as3"
        ]
  ```

  

  1-2. 컨트롤러 재 시작

  ```
  kubectl apply -f f5-k8s-bigip-ctlr.yaml
  ```



2. 오케스트레이션 모드

   | –agent option | Description                                                  |
   | :------------ | :----------------------------------------------------------- |
   | as3           | BIG-IP 오케스트레이션을 위해 AS3을 구현합니다. <br />이 옵션은 `<partition>` _AS3으로 추가 파티션을 만듭니다. <br />`<partition> `이름은 `–bigip-partition = <name>` 인자로 제공됩니다. |
   | cccl          | BIG-IP 오케스트레이션을위한 공통 컨트롤러 코어 라이브러리를 구현합니다. (기본 설정) |



* 지원되는 Kubernetes Ingress 기능

  * 단일 서비스
  * Simple Fanout
  * 이름 기반 가상 호스팅
  * TLS

* 지원하는 주석

  ```
  kubernetes.io/ingress.class
  virtual-server.f5.com/balance
  virtual-server.f5.com/health
  virtual-server.f5.com/ip
  virtual-server.f5.com/serverssl
  virtual-server.f5.com/http-port
  virtual-server.f5.com/https-port
  virtual-server.f5.com/ssl-redirect
  virtual-server.f5.com/allow-http
  virtual-server.f5.com/rewrite-app-root
  virtual-server.f5.com/rewrite-target-url
  ```

  



### 15. AS3 오버라이드

* 사용자 정의 config map과 함께 AS3을 사용 시

  * 기존 Kubernetes 리소스에 영향을 주지 않고 기존 Big-IP configuration 변경 가능
  * 기존 구성을 덮어 쓰거나 삭제할 필요없이 기존 BIG-IP 구성을 점진적으로 수정 가능

  

* AS3  오버라이드 기능을 이용할 시, 다음 명령을 실행해야 합니다.

  * `--override-as3-declaration=<namespace>/<user_defined_configmap_name>`
  * 실제 실행 예
  * ![../_images/as3-override.png](https://clouddocs.f5.com/containers/v2/_images/as3-override.png)

  * 설정하지 않으면 NULL 값 처리됨



* AS3 오버라이드 기능으로 가상 서버의 주소를 변경하는 예제

  * 가상서버 이름 :  `ingress_172_16_3_23_80`

  * 대상 IP : `172.16.3.23`

    ![../_images/as3-override-virtual-server-address.png](https://clouddocs.f5.com/containers/v2/_images/as3-override-virtual-server-address.png)

  * 사용자 정의 config map의 메타 데이터 섹션의 다음의 데이터를 제공

    * 이름 : 사용자 정의 config map의 이름

    * 네임 스페이스 : Deployment 네임 스페이스

    * 템플릿 : AS3과 동일한 계층 구조여야 함.

    * ```yaml
      kind: ConfigMap
      apiVersion: v1
      metadata:
       name: <configmap_name>
       namespace: <namespace>
      data:
       template: |
          {
              "declaration": {
                  "<partition_name>": {
                      "<application_name>": {
                          "<component_name>": {
                              "": [
                                  "<altered_value>"
                              ]
                          }
                      }
                  }
              }
          }
      ```

    

    * 가상 서버 IP 주소를 변경하기 위한 사용자 정의 config map

    * ```
      kind: ConfigMap
      apiVersion: v1
      metadata:
       name: example-vs
       namespace: default
      data:
       template: |
          {
              "declaration": {
                  "test_AS3": {
                      "Shared": {
                          "ingress_172_16_3_23_80": {
                              "virtualAddresses": [
                                  "172.16.3.111"
                              ]
                          }
                      }
                  }
              }
          }
      ```

  * 명령을 실행하여 구성 맵을 작성

    * `#kubectl create configmap <config_map_name>`

    * kubectl create configmap 명령을 사용하여 구성 맵을 작성하는 다른 세 가지 방법

      * 1. 전체 디렉토리의 내용을 사용

      ```
      #kubectl create configmap my-config --from-file=./my/dir/path/
      ```

      * 2. 파일 또는 특정 파일 세트의 내용을 사용

      ```
      #kubectl create configmap my-config --from-file=./my/file_name.json
      ```

      * 3. 명령 행에 정의 된 리터럴 키-값 쌍을 사용

      ```
      #kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2
      ```

      

  * 예제 실행 결과 화면

    ![../_images/as3-override-result.png](https://clouddocs.f5.com/containers/v2/_images/as3-override-result.png)





### 16. AS3을 사용하는 TLS 1.3 버전 지원

> - TLS 1.3을 사용하는 경우 암호 그룹을 포함해야 합니다. 
>   - 암호 그룹을 사용에 TLS 1.3이 필요한 것은 아닙니다.
>   -  암호 그룹의 기본값은 `f5-default` 입니다.
> - TLS 1.2를 사용하는 경우 암호 선택을 위해 암호 문자열을 지정할 수 있습니다.
> - *ciphers*(암호)와 *cipher group*(암호 그룹) 은 상호 배타적이므로 하나만 사용하십시오.



* CIS에서 TLS 1.3 구성

  1. 다음 명령을 사용하여 BIG-IP에서 TLS 버전이 활성화되도록 구성

  * 기본값 :  `1.2`

  ```
  --tls-version=<1.2 or 1.3>
  ```

  2. BIG-IP 시스템에서 암호 그룹을 구성하고 다음 명령을 사용하여 참조합니다. 

  * 기본 경로:  `/Common/f5-default`

  ```
  --cipher-group=<Complete path to BIG-IP Cipher Group>
  ```

  3. 콜론으로 구분 된 값과 함께 다음 명령을 사용하여 암호화 스위트 선택 문자열을 구성합니다.

  * 예 : `ECDHE_ECDSA:ECDHE`.
  * 기본값 :  `DEFAULT`

  ```
  --ciphers=<colon separated values. eg: ECDHE_ECDSA:ECDHE>
  ```

