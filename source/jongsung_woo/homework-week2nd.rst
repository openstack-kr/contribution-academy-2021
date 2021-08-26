==========================
Homework in Week 2nd
==========================

--------
Homework
--------

1. CLI와 친해지기
    4가지 방법은 팀 블로그의 본인 이름 폴더에 과제 폴더를 만든 뒤, "2주차 과제 - CLI와 친해지기" 이라는 제목으로 PR을 올려주세요.

    1. cirros image로 인스턴스 생성을 cli로 해보기
        :code:`openstack server create` ← 기본 명령어 + 옵션
    2. ubuntu 이미지를 받고, root password를 설정한 다음 cli로 이미지 등록한 후 인스턴스 생성하고 접속까지 하기
    3. cli로 floating ip 생성 후 인스턴스에 할당, 해제 해보기
        ubuntu image: `focal-server-cloudimg-amd64.img <https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img>`_
        
        만약 root password 설정을 못하겠다면, 인스턴스 생성 후 로그인 창 까지만 떠도 OK
    4. 10.8.0.0/24 네트워크를 만들고 public network와 연결하는 과정을 cli로 해보기(optional)

2. openstack server list 명령어 동작 원리 파악
    openstack server list 는 환경변수에 등록된 프로젝트의 모든 인스턴스를 출력하는 명령어입니다. python-openstackcli 의 코드 내부에서 이 명령어를 처리하기 위한 절차를 분석 후 자유롭게 보고서로 작성해주세요.

    - 주요 분석 포인트
        - 인자로 입력받은 :code:`server list` 를 어떻게 구별해내는가?
        - :code:`server list` 라는 명령어를 처리하는 파일은 무엇인가?
        - openstackcli 는 어떻게 nova api 주소를 알아내나요?
        - nova 의 어떤 API를 호출하여 결과를 받아오나요? (어떤 URI 를 호출하나요?)
        - 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?

각자 분석한 내용은 팀 블로그의 본인 이름 폴더에 과제 폴더를 만든 뒤, "2주차 과제 - openstack server list 명령어 동작 원리 파악" 이라는 제목으로 PR을 올려주세요.


------------------------
Background Knowledge
------------------------


OpneStack CLI의 기본 구조와 인증
=====================================

OpenStack CLI의 명령어 기본 구조는 아래와 같다.

.. code-block:: bash

    openstack [<global-options>] <command> [<command-arguments>]
    openstack help <command>
    opentsack --help

OpenStack CLI를 사용하기 위해선 인증 과정을 거쳐야 하는데, keystoneclient library에서 제공하는 token, username/password, openid 인증 방식을 사용할 수 있다.

일부 기능의 경우 요구하는 인증 방식이 있어 참고가 핑료하다. 인증관련된 기본적인 테스트를 위해 현재 프로젝트 리스트를 가져오는 :code:`project list` 명령어를 실행해서 테스트를 해볼 수 있다.

.. code-block:: bash

    openstack \
    --os-user-domain-name default \
    --os-username admin \
    --os-password secret \
    --os-auth-url http://your.ip/identity \
    project list

    `--os-user-domain-name`에 들어갈 값과 `--os-auth-url`의 값은 `/etc/openstack/clouds.yaml` 파일을 참고하면 된다.

:code:`openstack` 명령어는 기본적으로 인증에 필요한 값들을 환경변수에서 먼저 읽어 사용하는데 devstack 소스 프로젝트의 :code:`./devstack/openrc`를 실행하면 별도 인증 관련 argument 없이 :code:`openstack` 명령어를 실행할 수 있는 환경변수를 모두 설정해준다. 아래와 같이 실행할 수 있다.

.. code-block:: bash

    source ./devstack/openrc admin default



devstack 소스 프로젝의의 :code:`./devstack/openrc` 파일을 실행해서 기본 인증


Glossary
=====================================

.. glossary::
    
    domain
      users, groups, projects가 포함되는 collection이다. group, project와 domain의 관계는 1:N 관계로 group, project는 하나의 domain에만 속할 수 있다.
    
    Project
      OpenStack에서 ownership의 기본 단위. OpenStack의 모든 리소스는 project에 속해야 하고 project는 domain에 속해야 한다.

    Image
      image, virtual machine image란 부팅가능한 os가 설치되어 있는 가상 디스크를 포함하고 있는 단일 파일을 의미한다. 이미지를 사용해서 클라우드상에서 가상머신의 인스턴스를 생성할 수 있다. :code:`openstack image list`

    Security Group
      네트워크 트래픽을 필터링하는 rule을 정한 집합이다. `openstack security group list`

    Cloud Image
      일반적인 os 이미지의 경우 언어 선택, 디스크 사이즈 선택, 네트워크 선택 등 다양한 요소들을 사용자가 설치하며 진행하게 된다. 하지만 이런 사용자와의 interaction은 클라우드 등 auto deploy 환경에서 걸림돌이 될 수 있다. 클라우드 이미지는 자동화된 시스템에 의해서 os가 자동으로 배포될 수 있도록 유저와의 interfaction을 없애고 os 설치에 필요한 값들을 미리 지정하고 디스크 사이즈 등의 필요한 요소들을 파라미터화, 자동화 해놓은 이미지이다.


2-1 CLI와 친해지기: cirros image로 인스턴스 생성을 cli로 해보기
===============================================================================================================

가상 머신을 생성하기 위한 CLI 명령어는 :code:`openstack server create` 명령어를 사용하면 되는데, 몇가지 argument를 필수로 입력해야 한다.

* flavor: 가상머신의 컴퓨팅 리소스(compute, memory, storage)의 preset이다. GUI 화면에서 인스턴스 생성 단계의 'Flavor'에 해당는 부분이다. :code:`flavor list` 명령어를 사용해서 id와 name을 확인할 수 있다.
    .. image:: images/server-create-flavor.png
        :width: 600
* image: 인스턴스 사용에 생성할 베이스 이미지. GUI 화면에서 인스턴스 생성 단계의 '소스'에 해당하는 부분이다. :code:`image list` 명령어를 사용해서 id와 name을 확인할 수 있다.
    .. image:: images/server-create-image.png
        :width: 600
* network : 인스턴스서서 사용할 네트워크. GUI 화면에서 인스턴스 생성 단계의 '네트워크'에 해당하는 부분이다. :code:`network list` 명령어를 사용해서 id와 name을 확인할 수 있다.
    .. image:: images/server-create-network.png
        :width: 600

위 정보를 취합해 아래 명령어를 완성할 수 있다.

.. code-block:: bash
    :linenos:

    openstack \
      server create \
      --flavor m1.nano \
      --image cirros-0.5.2-x86_64-disk \
      --network private \
      cirros-test    # 생성할 인스턴스의 이름


2-2 ubuntu 이미지를 받고, root password를 설정한 다음 cli로 이미지 등록한 후 인스턴스 생성하고 접속까지 하기
===============================================================================================================

ubuntu 클라우드 이미지 배포판을 다운로드받는다.

.. code-block:: bash
    :linenos:

    wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img

클라우드 이미지의 설정을 수정할 수 있는 :code:`virt-customize` 툴을 다운로드, 사용해서 이미지의 root 비밀번호를 변경한다.

.. code-block:: bash
    :linenos:

    apt-get install libguestfs-tools
    sudo virt-customize -a focal-server-cloudimg-amd64.img --root-password password:secret

이전 :code:`2-1` 과제에서 사용했던 :code:`server create` 명령어를 사용하기 전에 비밀번호를 변경했던 이미지 파일을 OpenStack에 등록하는 절차가 필요하다. 이미지 등록은 :code:`image create` 명령어를 사용해서 할 수 있고 현재 이미지 리스트는 :code:`image list` 명령어를 사용해 볼 수 있다.

.. code-block:: bash
    :linenos:
    
    openstack \
      image create \
      --file focal-server-cloudimg-amd64.img \      # 업로드 할 로컬 이미지 파일 path
      --public \           # 해당 이미지 공개 범위
      --progress \        # 이미지 업로드 진행 상태를 %로 보여준다
      ubuntu.focal-server-cloudimg-amd64-root-pw-is-secret     # 사용할 이미지 네이밍

위 과정을 통해서 생성된 이미지로 아래와 같이 가상머신 인스턴스를 생성할 수 있다.

.. code-block:: bash
    :linenos:

    openstack \
      server create \
      --flavor ds512M \
      --image ubuntu.focal-server-cloudimg-amd64-root-pw-is-secret \
      --network private \
      ubuntu20  # 인스턴스의 이름

인스턴스를 생성할 때 :code:`2-1` 과제에 사용했던 :code:`m1.nano` flavor를 사용하면 아래와 같은 에러를 만날 수 있다. 각 이미지가 os로 설치되어 인스턴스화되기 위해 요구되는 최소한의 컴퓨팅 스펙이 있을 수 있으니 확인이 필요하다.

.. code-block:: bash
    :linenos:

    message : Build of instance 6293e308-5db8-40ff-aa86-7e7cadcca6fd aborted: Flavor\'s disk is too small for requested image. Flavor disk is 1073741824 bytes, image is 2361393152 bytes.
    code : 500
    details
    Traceback (most recent call last): File................


2-3 cli로 floating ip 생성 후 인스턴스에 할당 / 해제 해보기
===============================================================================================================

floating ip를 생성하고 이전 :code:`2-2` 과제에서 생성한 :code:`ubuntu20` 인스턴스에 할당할 것이다.

가상머신 인스턴스는 기본적으로 OpenStack에서 생성한, 가상머신 인스턴스끼리 통신할 수 있는 네트워크망을 제공한다. 이를 보통 :code:`내부`라고 불리우며, 외부와 통신을 하기 위해 외부에 속한 ip와 인스턴스를 연결해주게 되는데 이를 `공식 문서 <https://docs.openstack.org/ocata/user-guide/cli-manage-ip-addresses.html>`_에서는 :code:`associate` / :code:`disassociate` floating ip라고 표현한다.

floating ip를 생성하기 전에 먼저 floating ip를 제공할 수 있는 :code:`pool`을 확인해야 한다. :code:`floating ip pool list` 명령어로 확인할 수 있다.

.. code-block:: bash
    :linenos:

    openstack \
      floating ip \
      pool list

floating ip는 :code:`floating ip create` 명령어로 생성할 수 있으며 인자로 pool의 이름을 기재한다. 현재 프로젝트의 모든 floating ip 리스트는 :code:`floating ip list` 로 확인할 수 있다. 

.. code-block:: bash
    :linenos:

    openstack \
      floating ip \
      create \
      public       # pool 이름

:code:`server add floating ip` 명령어를 사용해서 생성한 floating ip를 associated 할 수 있다. disassociate 명령어는 :code:`server remove floating ip` 로 할 수 있다. 가상머신에서 disassociate된 floating ip는 인스턴스 연결만 해제되지 floating ip 자체는 삭제되지 않고 다시 다른 인스턴스에 붙일 수 있는 상태가 된다. floating ip를 삭제하기 위해선 :code:`floating ip delete` 로 삭제해주어야 한다.

.. code-block:: bash
    :linenos:

    openstack \
      server add \
      floating ip \
      ubuntu20 \                # floating ip를 associated할 인스턴스 이름
      192.168.0.100          # associate 시킬 floating ip의 주소 또는 id