openstack, k8s, ceph 삭제 가이드
==============================================


만약, openstack 설치 시 문제가 발생하여 openstack만 재설치를 하고싶다면 
------------------------------------------------------------------------

다음과 같이 openstack을 삭제할 수 있습니다.


::

   [clex@control1 burrito-1.2.4-rc.0]$ ./scripts/clean_openstack.sh
   Enter machine Hostname: control1
   persistentvolumeclaim "ingress-client-body-vol-ingress-0" deleted
   persistentvolumeclaim "ingress-client-body-vol-ingress-1" deleted
   configmap "ingress" deleted
   configmap "kube-root-ca.crt" deleted
   secret "ingress-tls-direct" deleted






만약, k8s 설치 시 문제가 발생하여 k8s만 재설치를 하고싶다면 
------------------------------------------------------------------------

다음과 같이 k8s를 삭제할 수 있습니다.



.. attention::

   ::
      단, openstack이 삭제되어야 합니다.
   


::

   ./scripts/clean_k8s.sh

   Are you sure you want to reset cluster state? Type 'yes' to reset your cluster. [no]:


질문에 yes라고 넣어야 합니다.





만약, ceph 설치 시 문제가 발생하여 ceph만 재설치를 하고싶다면 
------------------------------------------------------------------------

다음과 같이 ceph를 삭제할 수 있습니다.



.. attention::

   ::
      단, openstack, k8s가 삭제되어야 합니다.




::

   $ ./scripts/clean_ceph.sh 