오류 수정사항
===============

burrito.sh 이슈
-------------------

* 영향 받는 버전: 1.2.0

The burrito.sh 스크립트가 실행되지 않습니다.

The burrito.yml playbook은 kubespray 원본 저장소에 제거되었지만 burrito.sh 스크립트는 여전히 kubespray 디렉토리에서 찾고 있습니다.

:download:`patch <../_static/00-patch-burrito-sh.txt>` 다운로드 링크를 클릭하여 <burrito_source_dir>/patches/ 디렉토리에 넣습니다.

패치를 적용합니다.

::

    $ cd <burrito_src_dir>
    $ ./scripts/patch.sh