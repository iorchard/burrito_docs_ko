오류 수정사항
===============

burrito.sh 이슈
-------------------

* 영향 받는 버전: 1.2.0

The burrito.sh 스크립트가 실행되지 않습니다.

The burrito.yml playbook은 kubespray 원본 저장소에 제거되었지만 burrito.sh 스크립트는 여전히 kubespray 디렉토리에서 찾고 있습니다.

아래는 수정 패치입니다.

::

   @@ -7,7 +7,7 @@
    OSH_INFRA_PATH=${CURRENT_DIR}/../openstack-helm-infra
    OSH_PATH=${CURRENT_DIR}/../openstack-helm
    BTX_PATH=${CURRENT_DIR}/../btx/helm
   -KUBESPRAY_PATH=${CURRENT_DIR}/../kubespray
   +TOP_PATH=${CURRENT_DIR}/..
    OVERRIDE_PATH=$HOME/openstack-artifacts
    
    declare -A path_arr=(
   @@ -67,7 +67,7 @@
        ansible-playbook --extra-vars=@vars.yml ${OFFLINE_VARS} \
            --extra-vars="{\"$KEY\": [\"${NAME}\"]}" \
            ${TAG_OPTS} \
   -        ${KUBESPRAY_PATH}/burrito.yml
   +        ${TOP_PATH}/burrito.yml
      popd
    }
    uninstall() {