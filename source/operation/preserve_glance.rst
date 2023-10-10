glance image PV 보존 방법
==========================


개요
--------

Burrito에서 기본 저장 백엔드가 NetApp인 경우,
Persistent Volume (PV)가 생성되고 glance 이미지를 저장하는 데 사용됩니다.



PV는 NetApp Trident CSI 드라이버에 의해 동적으로 생성되며 저장 클래스 설정에서 상속된 reclaim 정책이 Delete로 설정되어 있으면, PV는 더 이상 파드에서 사용되지 않을 때 삭제됩니다.
따라서 glance를 제거하면 PV도 사라집니다.

우리는 glance가 삭제되더라도 PV를 보존하려 합니다.
그리고 glance를 다시 설치할 때 이미 업로드한 이미지를 재사용할 수 있습니다.



다음은 가이드입니다.

glance image Persistent Volume 보존 방법
------------------------------------------------

glance 이미지 PV의 reclaim 정책은 Delete입니다.

::

    root@btx-0:~# k get pv pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
    pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1   500Gi      RWX            Delete           Bound    openstack/glance-images   netapp                  25h

PV를 변경하여 Retain으로 설정합니다.

::

    root@btx-0:~# k patch pv pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1 -p \
        '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
    persistentvolume/pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1 patched

PV의 reclaim 정책 Retain으로 변경됩니다.

::

    root@btx-0:~# k get pv pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
    pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1   500Gi      RWX            Retain           Bound    openstack/glance-images   netapp                  25h

glance를 제거합니다.

::

    $ ./scripts/burrito.sh uninstall glance

glance-images PVC는 사라지지만 PV는 여전히 존재합니다.

::

    root@btx-0:/# k get pv pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                     STORAGECLASS   REASON   AGE
    pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1   500Gi      RWX            Retain           Released   openstack/glance-images   netapp                  25h

PV의 상태는 Released로 변경됩니다.

claimRef의 resourceVersion 및 uid를 null로 패치합니다.

::

    root@btx-0:~# k patch pv pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1 -p \
        '{"spec":{"claimRef": {"resourceVersion": null, "uid": null}}}'
    persistentvolume/pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1 patched

glance를 설치할 때 PV를 재사용하고 싶다면 적절한 YAML manifest 파일 (pvc-images.yaml)을 편집합니다.

::

    [burrito]$ git diff openstack-helm/glance/templates/pvc-images.yaml
    diff --git a/openstack-helm/glance/templates/pvc-images.yaml b/openstack-helm/glance/templates/pvc-images.yaml
    index 86a3b47..658c1ec 100644
    --- a/openstack-helm/glance/templates/pvc-images.yaml
    +++ b/openstack-helm/glance/templates/pvc-images.yaml
    @@ -26,5 +26,6 @@ spec:
         requests:
           storage: {{ .Values.volume.size }}
       storageClassName: {{ .Values.volume.class_name }}
    +  volumeName: pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1
     {{- end }}
     {{- end }}

'volumeName: <PV 이름>'를 추가하여 PVC가 PV를 사용하도록 합니다.

glance를 설치합니다.

::

    $ ./scripts/burrito.sh install glance
    
그리고 PV가 사용되는 것을 확인합니다.

::

    root@btx-0:/# k get pvc glance-images
    NAME            STATUS   VOLUME                                     CAPACITY ACCESS MODES   STORAGECLASS   AGE
    glance-images   Bound    pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1   500Gi   RWX            netapp         1m
    root@btx-0:/# k get pv |grep glance-images
    pvc-b0f01f0f-fab5-4c0f-b549-89f47d819cf1   500Gi      RWX     Retain Bound    openstack/glance-images                       netapp         26h

glance-images PVC는 보존된 PV를 사용합니다.