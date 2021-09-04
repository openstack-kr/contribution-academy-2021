===============
CLI 와 친해지기
===============
-----
목표
-----
  - GUI 로 했던 모든 과정을 CLI 로 하는 방법을 찾아보면서 CLI 도구와 친해지는 시간을 갖습니다.
  - 아래의 네 가지 과정을 어떻게 수행해야할 지 스스로 학습하여 구성해본 뒤, 문서로 정리합니다.

-----

`<Note>`
  OpenStack 실행 후 ``help`` 를 입력하면 사용 가능한 명령어 목록을 볼 수 있다.
  각각의 명령어들에 대해 어떤 옵션 값을 줄 수 있는지 확인한 후 아래의 과정대로 진행하였다.

--------------------------------------------
cirros image로 인스턴스 생성을 cli로 해보기
--------------------------------------------

1. ``image list`` 명령어로 이미지 목록을 확인한다.

  .. image:: ../images/week2/image_1.png 

2. ``network list`` 명령어로 현재 네트워크 목록을 확인한다.

  .. image:: ../images/week2/image_2.png

3. 확인한 값들을 옵션으로 넣어 인스턴스를 생성한다.

  .. code::

    server create --flavor m1.nano --image <image ID> --nic net-id=<network ID> --availability-zone nova <server name>
    ex) server create --flavor m1.nano --image 69e0e618-8038-4f2c-8889-11112c0cbd94 --nic net-id=private --availability-zone nova demo2

  .. image:: ../images/week2/image_3.png

-----

---------------------------------------------------------------------------------------------------
ubuntu 이미지를 받고, root password를 설정한 다음 cli로 이미지 등록한 후 인스턴스 생성하고 접속하기
---------------------------------------------------------------------------------------------------

(1) ``devstack/files`` 경로에 ubuntu 이미지를 다운받는다.

  .. code::

    wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img

(2) ``libguestfs-toos`` 패키지를 설치한다.

  .. code::

    sudo apt install libguestfs-tools

(3) root password 를 ubuntu 로 설정한다.

  .. code::

    sudo virt-customize -a focal-server-cloudimg-amd64.img --root-password password:ubuntu

(4) 위의 이미지 파일을 openstack 에 등록해준다.

  .. code::

    image create <image name> --file <file name> --disk-format raw —container-format=bare --public
    ex) image create "ubuntu" --file focal-server-cloudimg-amd64.img --disk-format raw —container-format=bare --public

(5) 이미지가 정상적으로 추가되었는지 ``image list`` 명령어를 사용하여 확인한다.

  .. image:: ../images/week2/image_4.png

(6) 위에서 추가한 ubuntu image 를 사용하여 인스턴스를 생성한다.

  .. code::

    server create --flavor <flavor> --image <image id> --nic net-id=<network id> --availability-zone nova <server name>
    ex) server create --flavor m1.small --image ubuntu --nic net-id=private --availability-zone nova demo3

  .. image:: ../images/week2/image_5.png

(7) 콘솔창에서 ``ID : root / PW : ubuntu`` 로 정상 로그인되는지 확인한다.

-----

--------------------------------------------------------
cli로 floating ip 생성 후 인스턴스에 할당 / 해제 해보기
--------------------------------------------------------

(1) floating ip를 생성해준다.

  .. code::

    floating ip create --floating-ip-address <floating ip> public

  .. image:: ../images/week2/image_6.png

(2) ``floating ip list`` 명령어로 floating ip 가 정상적으로 생성되었는지 확인한다.

  .. image:: ../images/week2/image_7.png

(3) 인스턴스에 생성한 floating ip를 할당해준다.

  .. code::

    server add floating ip <server name> <floating ip>

  .. image:: ../images/week2/image_8.png

(4) 인스턴스에 할당한 floating ip를 해제해본다.

  .. code::

    server remove floating ip <server name> <floating ip>

  `<Note>`
    floating ip delete 명령어를 사용할 경우 인스턴스에 할당한 floating ip 를 해제하는 것이 아닌 생성한 floating ip 자체가 삭제된다.

-----

----------------------------------------------------------------------------
10.8.0.0/24 네트워크를 만들고 public network와 연결하는 과정을 cli로 해보기
----------------------------------------------------------------------------

(1) network 를 생성해준다.

  .. code::

    network create --availability-zone-hint nova <network name>

  .. image:: ../images/week2/image_10.png

(2) 위에서 생성한 네트워크에 연결할 서브넷을 생성한다.

  .. code::

    subnet create —gateway <gateway> --network <network> —subnet-range <subnet-range> <name>
    ex) subnet create --gateway 10.8.0.1 --network network1 --subnet-range 10.8.0.0/24 network1-subnet

  .. image:: ../images/week2/image_11.png

(3) 라우터에 인터페이스를 추가해준다.

  .. code::

    router add subnet <router> <subnet>
    ex) router add subnet router1 network1-subnet

  .. image:: ../images/week2/image_12.png
