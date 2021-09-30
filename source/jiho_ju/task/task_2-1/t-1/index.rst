1. cirros image로 인스턴스 생성을 CLI로 해보기
==========================================================


.. toctree::

OpenStack 환경변수 설정
------------------------
OpenStack Command 를 사용하기 위해 환경변수를 세팅해줘야 한다.

- 환경변수 설정을 안해주었을 경우

.. warning::
    Missing value auth-url required for auth plugin password 에러? 발생

따라서 **환경변수에 openstack authentication을 추가해줘야 한다.**

devstack 은 환경변수를 계정에 맞게 설정해주는 스크립트를 제공해준다. (`ERROR`_)

.. code-block:: bash

   source openrc admi(사용자)n admin(프로젝트이름)

환경변수에 필요한 내용을 admin 사용자의 권한으로 읽어오고 admin 프로젝트를 default 로 바라보게 한다.
그 결과 openstack command (예: openstack server list) 가 잘 작동하는 것을 볼 수 있다.

OpenStack instance 생성
------------------------

.. code-block:: bash

   openstack server create --image=[가지고 있는 이미지] --flavor=[사양] --network=[public/private/shared] instance_name
   openstack [server, flavor, network] list # (server, flavor, image 등) list 출력

----------------------

ERROR
------------------------

CryptographyDeprecationWarning: int_from_bytes is deprecated, use int.from_bytes instead 에러
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

openstackclient 를 사용하기 위해 openrc 파일을 사용해 다음과 같은 명령어를 수행

.. code-block:: bash

   source openrc admin admin

그러자 위와 같은 에러가 발생,,,

원인
    **an upstream bug with twisted's dependency**

해결방안
    - crytography < 3.4 와 같은 옛날 버전의 crytography 를 설치해야 한다.
    - 이 버전은 twisted's dependency를 만족하고 deprecation warings 를 throw 하지 않는다.
    - cd /opt/stack/devstack 에서 **pip install cryptography==3.3** 명령어를 수행하니 더이상 에러가 발생하지 않는다.

version 3.3을 선택한 이유는 3.4 이하 버전에서 가장 최신 버전이라 선택

----------------------

참조
------

- `<https://seungjuitmemo.tistory.com/170>`_
- `<https://velog.io/@dojun527/OpenStack-CLI-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0>`_
