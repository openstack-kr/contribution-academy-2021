2주차 과제 : CLI 와 친해지기
==========================================================


공식 문서

https://docs.openstack.org/wallaby/user/

1) cirros image로 인스턴스 생성을 cli로 해보기

`openstack server create` ← 기본 명령어 + 옵션



2) ubuntu 이미지를 받고, root password를 설정한 다음 cli로 이미지 등록한 후 인스턴스 생성하고 접속까지 하기

- ubuntu image: https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
- 만약 root password 설정을 못하겠다면, 인스턴스 생성 후 로그인 창 까지만 떠도 OK

.. code-block::

    # image root password
    sudo apt install libguestfs-tools
    virt-customize -a 이미지파일 --root-password password:비밀번호
    virt-customize -a focal-server-cloudimg-amd64.img  --root-password password:1234

    # image 등록
    openstack image create --disk-format qcow2 --file ./focal-server-cloudimg-amd64.img ubuntu-20.04-lsh

    # cli 로 image 생성
    openstack server create --flavor m1.small --image ubuntu-20.04-lsh --nic net-id=ff134353-24c2-43fb-a943-48e4adf08dca --nic net-id=1d70c0ca-74eb-49d4-915d-f7c294436939 demo-ubuntu

    # console 접속 : root / 1234




3) cli로 floating ip 생성 후 인스턴스에 할당 / 해제 해보기

.. code-block::

    openstack floating ip show <floating-ip>

    # 네트워크 정보 확인
    openstack network list
    openstack network show fe1a50fe-50a1-4d29-9cc9-ff12f4d0085c

    # floating ip 는 router : external 설정된 network 에 할당 가능하다. => subnet 의 원리
    # floating ip 생성
    openstack floating ip create --description demo-floating-ip fe1a50fe-50a1-4d29-9cc9-ff12f4d0085c
    openstack floating ip create public


    # floating ip 연결
    openstack server add floating ip <INSTANCE_NAME_OR_ID> <FLOATING_IP_ADDRESS>
    openstack floating ip set --port <port> <floating-ip>
    openstack floating ip set --port 03632bb7-b8fe-490a-b91e-52f28495c3b0 --fixed-ip-address 10.0.0.18 192.168.100.155

    # 확인
    openstack floating ip list

    # floating ip 해제
    openstack server remove floating ip demo-ubuntu 192.168.100.155


    ref. https://docs.openstack.org/python-openstackclient/latest/?_ga=2.206237395.152728548.1629727800-574995788.1628140278

    ''' fail 한 명령어
    openstack floating ip unset --port 192.168.100.155
    openstack floating ip unset 192.168.100.155


4) 10.8.0.0/24 네트워크를 만들고 public network와 연결하는 과정을 cli로 해보기 (optional)


