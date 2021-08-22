CLI와 친해지기
================

gui로 했던 모든 과정을 cli로 하는 방법을 찾아보면서 cli 도구와 친해지는 시간을 갖습니다.

0) devstack 접속하여 Path 설정하기
--------------------------
#. ssh로 devstack을 설치한 터미널에 접속한다.
    * `ssh -i <pem key> <user@hostname>`
#. devstack을 설치한 계정으로 접근한다.
    * `sudo -u stack -i`
#. 환경변수 셋팅이 필요하다. 환경변수에 하나하나 값을 넣어주거나 스크립트를 사용하여 업데이트해준다. 필요한 환경변수는 아래와 같다.
    * OS_PROJECT_NAME
    * OS_TENANT_NAME
    * OS_USERNAME
    * OS_PASSWORD
    * OS_REGION_NAME
    * OS_AUTH_URL
    * OS_AUTH_TYPE
#. devstack에서 제공하는 스크립트는 아래의 명령어로 실행할 수 있다.
    * `source ~/devstack/openrc <계정이름> <프로젝트이름>`
    * `source ~/devstack/openrc admin admin`


1) cirros image로 인스턴스 생성을 cli로 해보기
----------------------------------------
* 인스턴스 생성은 server의 name, flavor의 종류, network 영역을 필수 argument로 두고 key_name 등의 optional도 지원하고 있다.
    .. code-block:: bash
        openstack server create
         --image cirros-0.5.2-x86_64-disk
         --flavor cirros256
         --network private
         cirros


2) ubuntu 이미지를 받고, root password를 설정한 다음 cli로 이미지 등록한 후 인스턴스 생성하고 접속까지 하기
-----------------------------------------
# ubuntu 이미지 파일 다운받기
    * `curl https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img ./ubuntu-20.04.img`
# 이미지 파일로 이미지 생성하기
    * `glance image-create --name ubuntu-20.04 --visibility private --disk-format qcow2 --container-format bare < ubuntu-20.04.img`
# 이미지로 instance 생성하기
    * `openstack server create --image ubuntu-20.04 --flavor m1.small --network private ubuntu`
#. Instance의 console 주소 찾기
    * `$ openstack console url show INSTANCE_NAME --xvpvnc`
#. root password 설정
#. Instance snapshot으로 image 등록하기
    .. code-block:: bash
        $ nova list;
        $ nova stop myInstance;
        $ nova image-create --poll <instance name> <snapshot name to create>;
#. snapshot으로 새로운 instance 만들기
    * `$ nova boot --flavor m1.small --image <snapshot name> <Instance name>`


3) cli로 floating ip 생성 후 인스턴스에 할당 / 해제 해보기
-------------------------------------
#. floating ip 생성하기
    * `floating openstack floating ip create --project admin --subnet public-subnet public`
#. floating ip 인스턴스에 할당하기
    .. code-block:: bash
        $ openstack floating ip list;
        $ openstack server list;
        $ openstack server add floating ip <Instance name> <ip-address>;
#. floating ip 인스턴스에서 해제하기
    * `openstack server remove floating ip <Instance name> <ip-address>`


4) 10.8.0.0/24 네트워크를 만들고 public network와 연결하는 과정을 cli로 해보기  (optional)
-------------------------------------------
#. private 네트워크 만들기
    * `$ openstack network create private-admin`
#. 10.8.0.0/24 서브넷 만들어 private에 할당하기
    * `$ openstack subnet create private-subnet --network private-admin --subnet-range 10.8.0.0/24`
#. 연결할 라으터 만들기
    * `$ openstack router create router-admin`
#. 라우터의 외부 게이트웨이로 public network 연결하기
    * `$ openstack router set <ROUTER ID> --external-gateway <Public network ID>`
#. 라우터와 private subnet 연결하기
    * `$ openstack router add subnet <ROUTER ID> <Private subnet ID>`


* Reference
------------------------------------
* ubuntu 20.04 image: https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
* image 만들고 인스턴스 만드는 법 : https://docs.openstack.org/ironic/rocky/install/configure-glance-images.html
* snapshot으로 image 만들기 : https://docs.openstack.org/mitaka/user-guide/cli_use_snapshots_to_migrate_instances.html
* floating ip 만들고 instance에 associated하기 : https://computingforgeeks.com/how-to-assign-floating-ip-to-openstack-instance/
* private network 만들어 public network와 연결하기 : https://docs.openstack.org/ocata/user-guide/cli-create-and-manage-networks.html
* rst 사용법 : https://sublime-and-sphinx-guide.readthedocs.io/en/latest/lists.html
