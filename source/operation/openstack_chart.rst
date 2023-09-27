openstack chart 배포 및 삭제
================================


burrito 스크립트를 이용해 openstack chart명을 넣어 배포와 삭제를 할 수 있습니다.

::


   [clex@control1 burrito-1.2.4-rc.0]$ ./scripts/burrito.sh
   Usage: ./scripts/burrito.sh <action> <chart_name>
      <action>: install, uninstall
      <chart_name>
      ingress, ceph-provisioners, mariadb, rabbitmq, memcached
      openvswitch, libvirt, keystone, glance, placement
      neutron, nova, cinder, horizon, barbican, btx



openstack chart 설명
-----------------------

* **Ingress**: Ingress 차트는 오픈스택 환경에서 외부에서 내부로 트래픽을 라우팅하는 데 사용됩니다. 주로 로드 밸런싱, SSL 종단 처리 및 URL 경로 기반 라우팅과 같은 기능을 제공합니다.

* **Ceph-provisioners**: Ceph-provisioners는 Ceph 스토리지 클러스터를 오픈스택에 연결하고 스토리지 자원을 프로비저닝하는 데 사용됩니다. Ceph는 분산 스토리지 시스템으로 오픈스택의 스토리지 백엔드로 자주 사용됩니다.

* **MariaDB**: MariaDB는 오픈스택의 데이터베이스 서비스로 사용됩니다. 이 데이터베이스는 오픈스택 서비스 및 구성 정보를 저장합니다.

* **RabbitMQ**: RabbitMQ는 오픈스택의 메시지 큐 서비스로 사용됩니다. 오픈스택 서비스 간의 통신 및 작업 스케줄링을 위한 메시지 큐 시스템입니다.

* **Memcached**: Memcached는 분산 캐시 시스템으로, 오픈스택에서 데이터 캐싱을 위해 사용됩니다. 빠른 데이터 액세스를 지원하며 성능 향상에 기여합니다.

* **OpenvSwitch**: OpenvSwitch는 가상 네트워크 스위치로 사용되며, 가상 머신 및 네트워크 리소스 관리에 필요한 네트워킹 기능을 제공합니다.

* **Keystone**: Keystone은 오픈스택의 인증 및 인가 서비스로, 사용자 및 서비스 인증 및 권한 관리를 담당합니다.

* **Glance**: Glance는 이미지 서비스로, 가상 머신 이미지 관리와 배포를 지원합니다.

* **Placement**: Placement는 리소스 프로비저닝 및 관리를 위한 서비스로, 가상 머신과 호스트 사이의 리소스 할당을 관리합니다.

* **Neutron**: Neutron은 네트워킹 서비스를 제공하는 오픈스택 서비스로, 가상 네트워크, 서브넷, 라우터 등을 관리합니다.

* **Nova**: Nova는 가상 머신 관리 서비스로, 가상 머신 인스턴스를 생성, 시작, 중지 및 삭제하는 기능을 제공합니다.

* **Cinder**: Cinder는 블록 스토리지 서비스로, 가상 머신에 대한 블록 레벨 스토리지를 제공하며, 볼륨 관리와 스냅샷 생성을 지원합니다.

* **Horizon**: Horizon은 오픈스택의 대시보드 인터페이스로, 사용자가 오픈스택 리소스를 시각적으로 관리할 수 있도록 합니다.

* **Barbican**: Barbican은 보안 관리 서비스로, 암호 키와 보안 인증서 관리를 지원합니다.

* **BTX**: BTX는 오픈스택 환경에서 사용되는 일부 특정한 차트나 서비스로, 더 구체적인 정보가 필요합니다. 일반적인 오픈스택 차트와는 다를 수 있습니다.



배포 및 삭제 ingress 예시
----------------------------

만약 ingress를 배포하고 싶다면 action = install, chart_name = ingress로 하여 playbook을 실행합니다.

::

   [clex@control1 burrito-1.2.4-rc.0]$ ./scripts/burrito.sh install ingress

   PLAY [Deploy ssh key pair in compute nodes.] ***********************************

   PLAY RECAP *********************************************************************
   control1                   : ok=16   changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   control2                   : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   control3                   : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   storage1                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   storage2                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   storage3                   : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

   ===============================================================================
   Gathering Facts --------------------------------------------------------- 6.49s
   burrito.openstack : OpenStack | deploy osh infra charts ----------------- 4.57s
   Gathering Facts --------------------------------------------------------- 2.82s
   burrito.openstack : OpenStack | make sure to set up control plane label --- 2.08s
   burrito.openstack : OpenStack | create openstack namespace -------------- 1.65s
   burrito.openstack : OpenStack | get cinder keyring ---------------------- 1.28s
   burrito.openstack : OpenStack | get admin keyring ----------------------- 1.03s
   burrito.openstack : OpenStack | templating osh infra values ------------- 1.01s
   burrito.openstack : OpenStack | make sure to set up compute node label --- 0.97s
   burrito.openstack : OpenStack | create openstack artifacts directory ---- 0.60s
   burrito.openstack : OpenStack | create dependency chart directory ------- 0.45s
   burrito.openstack : OpenStack | symlink helm-toolkit dependency chart --- 0.42s
   burrito.openstack : OpenStack | add ceph-provisioners to osh_infra_charts if ceph in storage_backends --- 0.12s
   burrito.openstack : OpenStack | set fact for admin keyring -------------- 0.10s
   burrito.openstack : OpenStack | add openvswitch to osh_infra_charts if neutron_ml2_plugin is ovs --- 0.09s
   burrito.openstack : OpenStack | debug ----------------------------------- 0.09s
   burrito.openstack : OpenStack | set fact for cinder keyring ------------- 0.09s
   ~/burrito-1.2.4-rc.0


만약 ingress를 삭제하고 싶다면 action = uninstall, chart_name = ingress로 하여 playbook을 실행합니다.

::

   [clex@control1 burrito-1.2.4-rc.0]$ ./scripts/burrito.sh uninstall ingress
   release "ingress" uninstalled
   pod "ingress-0" deleted
   pod "ingress-1" deleted
   pod "ingress-error-pages-7dd65557f8-pld6n" deleted
