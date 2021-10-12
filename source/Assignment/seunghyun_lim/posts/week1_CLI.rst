=======================
2week - CLI와 친해지기
=======================

gui로 했던 모든 과정을 cli로 하는 방법을 찾아보면서 cli 도구와 친해지는 시간을 갖습니다.

0) devstack 접속하여 Path 설정하기
-------------------------------------
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

.. code-block:: bash

    source ~/devstack/openrc <account_name> <project name>
    source ~/devstack/openrc admin admin



1) cirros image로 인스턴스 생성을 cli로 해보기
----------------------------------------------------
* 인스턴스 생성은 인스턴스 이름, flavor의 종류, 네트워크 영역을 필수로 입력해주어야 한다.

.. code-block:: bash

    stack@lsh-wallaby:~/devstack$ openstack server create
                                    --image cirros-0.5.2-x86_64-disk
                                    --flavor cirros256
                                    --network private
                                    cirros

    +--------------------------------------+--------+--------+--------------------------------------------+
    | ID                 | Name   | Status | Networks          | Image                       | Flavor     |
    +--------------------------------------+--------+--------+--------------------------------------------+
    | da139f9c-90f3-4ff0 | cirros | ACTIVE | private=10.0.0.44 | cirros-0.5.2-x86_64-disk    | cirros256  |
    +--------------------------------------+--------+--------+--------------------------------------------+


2) ubuntu 이미지를 받고, root password를 설정한 다음 cli로 이미지 등록한 후 인스턴스 생성하고 접속까지 하기
---------------------------------------------------------------------------------------------------------------------
#. ubuntu 이미지 파일 다운받기
    * `$ curl https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img ./ubuntu-20.04.img`
#. root password 설정
    * 내려받은 이미지에 libguestfs-tools 패키지를 설치하여 root password를 설정합니다.
    * `sudo virt-customize -a ubuntu-20.04.img --root-password password:secret`

.. code-block:: bash

    stack@lsh-wallaby:~/glance/images$ sudo virt-customize -a ubuntu-20.04.img --root-password password:secret
        [   0.0] Examining the guest ...
        [  28.1] Setting a random seed
        [  28.2] Setting passwords
        [  40.3] Finishing off

#. 비밀번호 설정한 이미지 파일로 이미지 생성하기
    * `$ glance image-create --name ubuntu-20.04 --visibility private --disk-format qcow2 --container-format bare < ubuntu-20.04.img`

.. code-block:: bash

    stack@lsh-wallaby:~$ glance image-create --name ubuntu-20.04 --visibility private --disk-format qcow2 --container-format bare < ubuntu-20.04.img
    +------------------+----------------------------------------------------------------------------------+
    | Property         | Value                                                                            |
    +------------------+----------------------------------------------------------------------------------+
    | checksum         | d0b47062e215e43565d211f82de09493                                                 |
    | container_format | bare                                                                             |
    | created_at       | 2021-08-22T18:52:43Z                                                             |
    | disk_format      | qcow2                                                                            |
    | id               | ce68a2bf-af1c-4de5-90b7-bc2e2abb300c                                             |
    | min_disk         | 0                                                                                |
    | min_ram          | 0                                                                                |
    | name             | ubuntu-20.04                                                                     |
    | os_hash_algo     | sha512                                                                           |
    | os_hash_value    | c647438cf8c4bb00425f91496c5dd82e2c7fe26ef4eeb8b1b2aa0788429bb542824479ff13ab0b62 |
    |                  | 826c26a3d4fcb26f750e0584d2fc62017c4ab3b393955d85                                 |
    | os_hidden        | False                                                                            |
    | owner            | f414466b249b41e097c4047dcbf11ac9                                                 |
    | protected        | False                                                                            |
    | size             | 565051392                                                                        |
    | status           | active                                                                           |
    | tags             | []                                                                               |
    | updated_at       | 2021-08-22T18:52:47Z                                                             |
    | virtual_size     | 2361393152                                                                       |
    | visibility       | private                                                                          |
    +------------------+----------------------------------------------------------------------------------+

#. 이미지로 instance 생성하기

.. code-block:: bash

    stack@lsh-wallaby:~$ openstack server create --image ubuntu-20.04 --flavor m1.small --network private ubuntu
    +-------------------------------------+-----------------------------------------------------+
    | Field                               | Value                                               |
    +-------------------------------------+-----------------------------------------------------+
    | OS-DCF:diskConfig                   | MANUAL                                              |
    | OS-EXT-AZ:availability_zone         |                                                     |
    | OS-EXT-SRV-ATTR:host                | None                                                |
    | OS-EXT-SRV-ATTR:hypervisor_hostname | None                                                |
    | OS-EXT-SRV-ATTR:instance_name       |                                                     |
    | OS-EXT-STS:power_state              | NOSTATE                                             |
    | OS-EXT-STS:task_state               | scheduling                                          |
    | OS-EXT-STS:vm_state                 | building                                            |
    | OS-SRV-USG:launched_at              | None                                                |
    | OS-SRV-USG:terminated_at            | None                                                |
    | accessIPv4                          |                                                     |
    | accessIPv6                          |                                                     |
    | addresses                           |                                                     |
    | adminPass                           | NUNGqRf2Civ4                                        |
    | config_drive                        |                                                     |
    | created                             | 2021-08-22T18:54:02Z                                |
    | flavor                              | m1.small (2)                                        |
    | hostId                              |                                                     |
    | id                                  | bf2bf205-6ea5-4064-b2ff-73f9a01c0d93                |
    | image                               | ubuntu-20.04 (ce68a2bf-af1c-4de5-90b7-bc2e2abb300c) |
    | key_name                            | None                                                |
    | name                                | ubuntu                                              |
    | progress                            | 0                                                   |
    | project_id                          | f414466b249b41e097c4047dcbf11ac9                    |
    | properties                          |                                                     |
    | security_groups                     | name='default'                                      |
    | status                              | BUILD                                               |
    | updated                             | 2021-08-22T18:54:02Z                                |
    | user_id                             | 1692d1df5ff943728fe1d5c317751d4e                    |
    | volumes_attached                    |                                                     |
    +-------------------------------------+-----------------------------------------------------+

#. 접속 체크

.. code-block:: bash

    ubuntu login: root
    Password:
    Welcome to Ubuntu 20.04.3 LTS (GNU/LINUX 5.4.0-81-generic x86_64)
    root@ubuntu:~#

3) cli로 floating ip 생성 후 인스턴스에 할당 / 해제 해보기
-----------------------------------------------------------------------------------------
#. floating ip 생성하기

.. code-block:: bash

    stack@lsh-wallaby:~$ openstack floating ip create --project admin --subnet public-subnet public
    +---------------------+--------------------------------------+
    | Field               | Value                                |
    +---------------------+--------------------------------------+
    | created_at          | 2021-08-22T19:05:04Z                 |
    | description         |                                      |
    | dns_domain          | None                                 |
    | dns_name            | None                                 |
    | fixed_ip_address    | None                                 |
    | floating_ip_address | 192.168.100.78                       |
    | floating_network_id | 694d6baf-2796-480f-8232-1f6947d4982d |
    | id                  | 23fc246c-af8a-46ce-96e1-026933581657 |
    | name                | 192.168.100.78                       |
    | port_details        | None                                 |
    | port_id             | None                                 |
    | project_id          | f414466b249b41e097c4047dcbf11ac9     |
    | qos_policy_id       | None                                 |
    | revision_number     | 0                                    |
    | router_id           | None                                 |
    | status              | DOWN                                 |
    | subnet_id           | f6c41fcb-2051-470c-9d0e-516eaa50c7e8 |
    | tags                | []                                   |
    | updated_at          | 2021-08-22T19:05:04Z                 |
    +---------------------+--------------------------------------+

#. floating ip 인스턴스에 할당하기

.. code-block:: bash

    $ openstack floating ip list
    $ openstack server list
    $ openstack server add floating ip <Instance name> <ip-address>

#. floating ip 인스턴스에서 해제하기
    * `openstack server remove floating ip <Instance name> <ip-address>`


4) 10.8.0.0/24 네트워크를 만들고 public network와 연결하는 과정을 cli로 해보기  (optional)
-------------------------------------------------------------------------------------------
#. private 네트워크 만들기
    * `$ openstack network create private-admin`
#. 10.8.0.0/24 서브넷 만들어 private에 할당하기
    * `$ openstack subnet create private-subnet --network private-admin --subnet-range 10.8.0.0/24`
#. 연결할 라우터 만들기
    * `$ openstack router create router-admin`
#. 라우터의 외부 게이트웨이로 public network 연결하기
    * `$ openstack router set <ROUTER ID> --external-gateway <Public network ID>`
#. 라우터와 private subnet 연결하기
    * `$ openstack router add subnet <ROUTER ID> <Private subnet ID>`


Reference
------------------------------------
* ubuntu 20.04 image: https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
* image 만들고 인스턴스 만드는 법 : https://docs.openstacg/mitaka/user-guide/cli_use_snapshots_to_migrate_instances.html
* floating ip 만들고 instance에 associated하기 : https:k.org/ironic/rocky/install/configure-glance-images.html
* snapshot으로 image 만들기 : https://docs.openstack.or//computingforgeeks.com/how-to-assign-floating-ip-to-openstack-instance/
* private network 만들어 public network와 연결하기 : https://docs.openstack.org/ocata/user-guide/cli-create-and-manage-networks.html
* rst 사용법 : https://sublime-and-sphinx-guide.readthedocs.io/en/latest/lists.html
