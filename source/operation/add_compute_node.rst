compute 노드 추가
===================


다음 내용은 이미 있는 burrito cluster에 compute 노드를 추가하는 가이드입니다.

새로운 compute node를 bon-compute2이고 IP 주소를 192.168.21.124라고 가정합니다.

모든 작업은 ansible deployer(control1)에서 수행됩니다.

새로운 compute node 정의
-----------------------------

bon-compute2(호스트 이름)과 IP(192.168.21.124)를 /etc/hosts에 정의합니다.

::

   $ sudo vi /etc/hosts
   192.168.21.124 bon-compute2

인벤토리 호스트에 새로운 compute node를 추가합니다.

::

   $ diff -u hosts.bak hosts
   --- hosts.bak        2023-02-20 13:54:45.365350417 +0900
   +++ hosts    2023-02-20 14:43:02.897660764 +0900
   @@ -1,6 +1,7 @@
    bon-controller ip=192.168.21.121 ansible_connection=local ansible_python_interpreter=/usr/bin/python3
    bon-compute ip=192.168.21.122 
    bon-storage ip=192.168.21.123 monitor_address=192.168.24.123 radosgw_address=192.168.24.123 
   +bon-compute2 ip=192.168.21.124

    # ceph nodes
    [mons]
   @@ -18,14 +19,16 @@
    [clients]
    bon-controller
    bon-compute
   +bon-compute2

    # kubernetes nodes
    [kube_control_plane]
    bon-controller

    [kube_node]
    bon-controller
    bon-compute
   +bon-compute2

    # openstack nodes
    [controller-node]
   @@ -36,6 +39,7 @@

    [compute-node]
    bon-compute
   +bon-compute2

새로운 노드 bon-compute2 네트워크 연결을 확인합니다.

::

   $ ./run.sh ping 
   bon-compute2 | SUCCESS => {
       "ansible_facts": {
           "discovered_interpreter_python": "/usr/libexec/platform-python"
       },
       "changed": false,
       "ping": "pong"
   }


새로운 노드 설치
-------------------------

bon-compute2에 대한 preflight playbook 실행합니다.

::

   $ ./run.sh preflight --limit=bon-compute2

ceph playbook 실행합니다.

::

   $ ./run.sh ceph --limit=bon-compute2

k8s cluster에 노드를 추가합니다.

::

   $ ./run.sh scale --limit=bon-compute2

k8s 노드에 추가되었는지 확인합니다.

::

   $ sudo kubectl get nodes
   NAME             STATUS   ROLES           AGE     VERSION
   bon-compute      Ready    <none>          3d15h   v1.24.8
   bon-compute2     Ready    <none>          3m39s   v1.24.8
   bon-controller   Ready    control-plane   3d15h   v1.24.8

tag=k8s-burrito,novakey-burrito 설정하여 burrito playbook 실행합니다.

::

   $ ./run.sh burrito --tags=k8s-burrito,novakey-burrito

compute 노드가 추가되었는지 확인합니다.

::

   root@btx-0:/# openstack compute service list
   +--------------------------------------+----------------+---------------------------------+----------+---------+-------+----------------------------+
   | ID                                   | Binary         | Host                            | Zone     | Status  | State | Updated At                 |
   +--------------------------------------+----------------+---------------------------------+----------+---------+-------+----------------------------+
   | e0a00939-3d0a-41d8-be9b-9dbb22ee5f11 | nova-scheduler | nova-scheduler-76c5874458-dlx8n | internal | enabled | down  | 2023-02-20T07:21:53.000000 |
   | 5d047fa1-0691-470a-803d-2df4a83dc1a3 | nova-conductor | nova-conductor-86c647ffdd-5l9md | internal | enabled | down  | 2023-02-20T07:21:53.000000 |
   | d7f9e8fc-13f5-4860-8573-116d09147850 | nova-compute   | bon-compute                     | nova     | enabled | up    | 2023-02-20T07:56:01.000000 |
   | 9b44d557-308e-4cd5-93c1-61843a2078da | nova-compute   | bon-compute2                    | nova     | enabled | up    | 2023-02-20T07:56:06.000000 |
   | 8f9aa838-0f4d-4029-b4df-9cbd89750723 | nova-scheduler | nova-scheduler-869cd8674d-7mcmp | internal | enabled | up    | 2023-02-20T07:56:03.000000 |
   | a1eae609-fa72-40f7-b7c4-300362a50fed | nova-conductor | nova-conductor-5c8f7fd658-6mbrp | internal | enabled | up    | 2023-02-20T07:56:04.000000 |
   +--------------------------------------+----------------+---------------------------------+----------+---------+-------+----------------------------+
   root@btx-0:/# openstack hypervisor list
   +----+---------------------+-----------------+----------------+-------+
   | ID | Hypervisor Hostname | Hypervisor Type | Host IP        | State |
   +----+---------------------+-----------------+----------------+-------+
   |  1 | bon-compute         | QEMU            | 192.168.21.122 | up    |
   |  2 | bon-compute2        | QEMU            | 192.168.21.124 | up    |
   +----+---------------------+-----------------+----------------+-------+

