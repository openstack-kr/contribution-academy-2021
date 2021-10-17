==========================================================
기본적인 CLI 사용하기
==========================================================

CLI 동작 원리
~~~~~~~~~~~~~~~~
오픈스택의 Horizon 또는 CLI 를 이용하여 인스턴스를 생성하면, Nova API 서버가 요청을 받아 Nova 에서는 Keystone 을 호출해 사용자 인증을 합니다.
이 요청이 성공하면 인스턴스가 생성될 호스트 서버를 확인하고 네트워크 아이피를 할당하고, Glance 를 호출하여 OS 이미지를 가져옵니다.
마지막으로 하이퍼바이저를 통해 인스턴스를 생성하게 됩니다. 이처럼 각각의 컴포넌트들은 유기적으로 REST API로 통신을 하며, 이러한 과정에 대한 상태값은 데이터베이스에 저장됩니다.

본 글에서는 이러한 과정을 CLI 를 통해 신규 인스턴스를 생성하여 기본적인 CLI 를 다뤄봅니다.

|


**인스턴스를 생성하기 위한 기본 정보**

 - 인스턴스에 할당할 Flavor 정보
 - 인스턴스의 Image
 - 인스턴스에 연결할 네트워크
 - 보안 그룹 정보
 - 인스턴스로 액세스하기 위한 키페어
 - 인스턴스명

|

인스턴스를 생성하기 위한 기본 정보를 조회하기 위한 명령어
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 - 인스턴스를 다루는 명령어는 **server** 명령을 이용합니다.
 - 인스턴스 목록을 조회하는 명령어로는 다음 명령으로 확인이 가능합니다.

 .. code-block:: none

    openstack server list

|

 - 인스턴스를 생성하기 위해서는 인스턴스에 할당할 리소스 정보가 필요합니다. 각 인스턴스에 할당 가능한 리소스 정보를 확인하는 명령어는 다음과 같습니다.
 - (참고: 오픈스택 공식 문서에서는 가장 작은 디폴트 Flavor는 인스턴스당 512MB 메모리를 소비하며, 4G보다 적은 메모리를 갖는 컴퓨트 노드 환경에서는 64MB만을 필요로 하는 m1.nano Flavor 로 생성할 것을 권장합니다.)

 .. code-block:: none

    openstack flavor list

|

 - 인스턴스를 생성하기 위해서는 인스턴스에 등록할 운영체제에 대한 이미지를 필요로합니다.
 - 아래 명령어는 등록된 이미지 목록을 확인할 수 있는 명령입니다.

 .. code-block:: none

    openstack image list

|

 - 인스턴스를 생성하기 위해서는 인스턴스에 네트워크 인터페이스(NIC)을 만들어 인스턴스 생성시 선택한 네트워크에 연결하여 호스트 아이피를 할당합니다.
 - 생성할 인스턴스에 연결할 네트워크를 지정하기 위해서 아래와 같은 명령으로 네트워크 정보를 확인합니다.

 .. code-block:: none

    openstack network list

|

 - 인스턴스 생성시 사용할 보안 정책을 지정합니다. 방화벽 및 오픈스택의 기본 방화벽 정책은 `이 링크 <https://docs.openstack.org/ko_KR/install-guide/firewalls-default-ports.html>`_ 에서 참고해주세요.
 - 오픈스택의 보안 정책인 시큐리티 그룹은 아래와 같은 명령으로 확인합니다.
 - (참고: 원격 접속을 위한 SSH 프로토콜의 기본 포트인 22번 포트와 ICMP 프로토콜은 허용되어 있지 않으므로, 필요시 정책을 등록합니다.)

 .. code-block:: none

    openstack security group list

|

 - 인스턴스에 접근하기 위한 키페어를 지정합니다. 대부분의 클라우드 이미지는 암호를 입력하여 세션을 연결했던 인증 방식 보다는 Public Key Authentication, 키페어를 이용하여 연결하는 방식을 권장합니다.
 - 인스턴스를 생성하기 전에 공개키를 생성하여 인스턴스에 등록하기 위해 키페어 목록을 확인합니다. 키페어 목록은 다음과 같은 명령으로 확인 가능합니다.
 - (참고: 공개키는 기존에 생성된 키를 사용할 수 있습니다.)

 .. code-block:: none

    openstack keypair list

|

인스턴스에 연결하기 위한 가상의 네트워크를 생성하는 방법
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

다음의 예는 인스턴스를 생성하기 위해 10.8.0.0/24 의 가상의 네트워크를 만드는 예제입니다.

 - 인스턴스를 생성하기 위한 가상의 네트워크를 생성하기 전, 기존에 생성된 가상 네트워크에 대한 모든 정보를 확인합니다.

 .. code-block:: none

    openstack network list

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack network list
    +--------------------------------------+---------+----------------------------------------------------------------------------+
    | ID                                   | Name    | Subnets                                                                    |
    +--------------------------------------+---------+----------------------------------------------------------------------------+
    | 4f68b05b-7620-46df-8a2f-5c5b7ff9a635 | shared  | 8c2c28cb-d6ed-419f-a6d1-fa571a34cdf5                                       |
    | 6ae865d8-8a2b-4e1f-aef7-5775f3203430 | private | 801b339a-4584-441b-a5d6-884daaf0c6d6, 82c1da22-6280-47da-9d47-735c510e1674 |
    | fccc03a4-16c7-425b-a163-0b58e7a01fdb | public  | 3fb8b87e-8440-4db2-a8f6-591115efaae1, 50dfb87a-7bfc-4890-8c01-8ed95f0c8de6 |
    +--------------------------------------+---------+----------------------------------------------------------------------------+

|

 - net1 이라는 이름을 가진 가상의 네트워크를 생성합니다.

 .. code-block:: none

    openstack network create net1

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack network create net1
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | UP                                   |
    | availability_zone_hints   |                                      |
    | availability_zones        |                                      |
    | created_at                | 2021-10-17T11:50:35Z                 |
    | description               |                                      |
    | dns_domain                | None                                 |
    | id                        | db6dafc1-8f4a-4079-a8d0-1c9c6e14d9c4 |
    | ipv4_address_scope        | None                                 |
    | ipv6_address_scope        | None                                 |
    | is_default                | False                                |
    | is_vlan_transparent       | None                                 |
    | mtu                       | 1450                                 |
    | name                      | net1                                 |
    | port_security_enabled     | True                                 |
    | project_id                | 1790d56ebff849be81e7af3ac06a33c5     |
    | provider:network_type     | vxlan                                |
    | provider:physical_network | None                                 |
    | provider:segmentation_id  | 256                                  |
    | qos_policy_id             | None                                 |
    | revision_number           | 1                                    |
    | router:external           | Internal                             |
    | segments                  | None                                 |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tags                      |                                      |
    | updated_at                | 2021-10-17T11:50:38Z                 |
    +---------------------------+--------------------------------------+

|

 - 네트워크의 서브넷 정보를 추가합니다.

 .. code-block:: none

    openstack subnet create net1-subnet --network net1 --subnet-range <network ip> --gateway <gateway ip> --dns-nameserver <DNS ip>

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack subnet create net1-subnet --network net1 --subnet-range 10.8.0.0/24 --gateway 10.8.0.1 --dns-nameserver 1.1.1.1
    +----------------------+--------------------------------------+
    | Field                | Value                                |
    +----------------------+--------------------------------------+
    | allocation_pools     | 10.8.0.2-10.8.0.254                  |
    | cidr                 | 10.8.0.0/24                          |
    | created_at           | 2021-10-17T11:57:13Z                 |
    | description          |                                      |
    | dns_nameservers      | 1.1.1.1                              |
    | dns_publish_fixed_ip | None                                 |
    | enable_dhcp          | True                                 |
    | gateway_ip           | 10.8.0.1                             |
    | host_routes          |                                      |
    | id                   | afc7bbf4-3f33-4352-b2fe-0d7adab19e70 |
    | ip_version           | 4                                    |
    | ipv6_address_mode    | None                                 |
    | ipv6_ra_mode         | None                                 |
    | name                 | net1-subnet                          |
    | network_id           | db6dafc1-8f4a-4079-a8d0-1c9c6e14d9c4 |
    | prefix_length        | None                                 |
    | project_id           | 1790d56ebff849be81e7af3ac06a33c5     |
    | revision_number      | 0                                    |
    | segment_id           | None                                 |
    | service_types        |                                      |
    | subnetpool_id        | None                                 |
    | tags                 |                                      |
    | updated_at           | 2021-10-17T11:57:13Z                 |
    +----------------------+--------------------------------------+

|

 - 생성한 서브넷과 네트워크 정보를 확인합니다. 올바르게 생성되었는지 확인합니다.

 .. code-block:: none

    openstack network list
    openstack subnet list

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack network list
    +--------------------------------------+---------+----------------------------------------------------------------------------+
    | ID                                   | Name    | Subnets                                                                    |
    +--------------------------------------+---------+----------------------------------------------------------------------------+
    | 4f68b05b-7620-46df-8a2f-5c5b7ff9a635 | shared  | 8c2c28cb-d6ed-419f-a6d1-fa571a34cdf5                                       |
    | 6ae865d8-8a2b-4e1f-aef7-5775f3203430 | private | 801b339a-4584-441b-a5d6-884daaf0c6d6, 82c1da22-6280-47da-9d47-735c510e1674 |
    | db6dafc1-8f4a-4079-a8d0-1c9c6e14d9c4 | net1    | afc7bbf4-3f33-4352-b2fe-0d7adab19e70                                       |
    | fccc03a4-16c7-425b-a163-0b58e7a01fdb | public  | 3fb8b87e-8440-4db2-a8f6-591115efaae1, 50dfb87a-7bfc-4890-8c01-8ed95f0c8de6 |
    +--------------------------------------+---------+----------------------------------------------------------------------------+

    stack@doa-wallaby-2:~/devstack$ openstack subnet list
    +--------------------------------------+---------------------+--------------------------------------+--------------------+
    | ID                                   | Name                | Network                              | Subnet             |
    +--------------------------------------+---------------------+--------------------------------------+--------------------+
    | 3fb8b87e-8440-4db2-a8f6-591115efaae1 | public-subnet       | fccc03a4-16c7-425b-a163-0b58e7a01fdb | 192.168.100.0/24   |
    | 50dfb87a-7bfc-4890-8c01-8ed95f0c8de6 | ipv6-public-subnet  | fccc03a4-16c7-425b-a163-0b58e7a01fdb | 2001:db8::/64      |
    | 801b339a-4584-441b-a5d6-884daaf0c6d6 | private-subnet      | 6ae865d8-8a2b-4e1f-aef7-5775f3203430 | 10.0.0.0/26        |
    | 82c1da22-6280-47da-9d47-735c510e1674 | ipv6-private-subnet | 6ae865d8-8a2b-4e1f-aef7-5775f3203430 | fd90:a103:66e::/64 |
    | 8c2c28cb-d6ed-419f-a6d1-fa571a34cdf5 | shared-subnet       | 4f68b05b-7620-46df-8a2f-5c5b7ff9a635 | 192.168.233.0/24   |
    | afc7bbf4-3f33-4352-b2fe-0d7adab19e70 | net1-subnet         | db6dafc1-8f4a-4079-a8d0-1c9c6e14d9c4 | 10.8.0.0/24        |
    +--------------------------------------+---------------------+--------------------------------------+--------------------+

|

 - 생성한 네트워크에 연결할 라우터 목록을 조회합니다.

 .. code-block:: none

    openstack router list

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack router list
    +--------------------------------------+---------+--------+-------+----------------------------------+-------------+-------+
    | ID                                   | Name    | Status | State | Project                          | Distributed | HA    |
    +--------------------------------------+---------+--------+-------+----------------------------------+-------------+-------+
    | 94743e4c-b771-453a-a0ec-555e9ca0ed5a | router1 | ACTIVE | UP    | 81731826a82f437b8ac0f7a793916ce9 | False       | False |
    +--------------------------------------+---------+--------+-------+----------------------------------+-------------+-------+

|

 - 라우터에 생성한 가상의 네트워크를 연결합니다.

 .. code-block:: none

    openstack router add subnet <router ID> <subnet ID>

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack router add subnet 94743e4c-b771-453a-a0ec-555e9ca0ed5a afc7bbf4-3f33-4352-b2fe-0d7adab19e70

|

인스턴스를 생성하기 위한 Ubuntu 20.04v 이미지 등록
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

가상의 네트워크를 생성하였으므로, 이제 인스턴스를 생성하기 위해서 우분투 이미지를 가져와 해당 이미지를 등록합니다.
본 챕터에서는 이미지를 등록하는 방법에 대한 내용을 다룹니다.

 - 이미지를 다운로드 받을 리포지터리는 다음과 같습니다.
 - 다운로드 받을 이미지를 선택하고, wget 혹은 curl 으로 서버에 이미지 파일을 가져옵니다.
 - URL: https://cloud-images.ubuntu.com/focal/current

 .. code-block:: none

    apt install wget
    wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack/images$ wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
    --2021-10-17 12:41:05--  https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
    Resolving cloud-images.ubuntu.com (cloud-images.ubuntu.com)... 91.189.91.124, 91.189.91.123, 2001:67c:1562::28, ...
    Connecting to cloud-images.ubuntu.com (cloud-images.ubuntu.com)|91.189.91.124|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 567476224 (541M) [application/octet-stream]
    Saving to: ‘focal-server-cloudimg-amd64.img’

    focal-server-cloudimg-amd64.img                100%[=================================================================================================>] 541.19M  15.2MB/s    in 38s

    2021-10-17 12:41:43 (14.4 MB/s) - ‘focal-server-cloudimg-amd64.img’ saved [567476224/567476224]

|

 - 클라우드 이미지를 다운로드 받은 후, 초기 패스워드를 설정하기 위해 libguestfs-tools 패키지를 설치합니다.
 - 이 패키지로 관리자 계정인 root 계정에 대한 초기 패스워드 설정이 가능합니다.

 .. code-block:: none

    sudo apt install libguestfs-tools

|

 - 이미지에 대한 초기 패스워드를 설정하는 작업은 sudo 권한으로 진행합니다.
 - 아래의 예제는 secret 으로 패스워드를 설정하는 예입니다.

 .. code-block:: none

    sudo virt-customize -a focal-server-cloudimg-amd64.img --root-password password:secret

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack/images$ sudo virt-customize -a focal-server-cloudimg-amd64.img --root-password password:secret
    [   0.0] Examining the guest ...
    [  16.4] Setting a random seed
    virt-customize: warning: random seed could not be set for this type of
    guest
    [  16.4] Setting passwords
    [  24.7] Finishing off

|

 - 패스워드 초기화를 한 후, 가져온 이미지 파일을 등록합니다.

 .. code-block:: none

    openstacak image create <image-alias> --file <image file name> --disk-format <disk format type> --container-format <format type> --액세스 권한

 .. code-block:: none

    stack@doa-wallaby-2:~/devstack/images$ openstack image create "ubuntu_20.04v"
    --file focal-server-cloudimg-amd64.img
    --disk-format qcow2 --container-format bare --public

|

 - 등록한 이미지 리스트를 조회합니다.

 .. code-block:: none

    openstack image list

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack/images$ openstack image list
    +--------------------------------------+--------------------------+--------+
    | ID                                   | Name                     | Status |
    +--------------------------------------+--------------------------+--------+
    | 29398b27-2654-4c81-9d75-e0f9db4a6059 | cirros-0.5.2-x86_64-disk | active |
    | 1cd3400c-d954-48b6-9e8a-8add12fa0b8c | ubuntu_20.04             | active |
    +--------------------------------------+--------------------------+--------+

|

생성할 인스턴스에게 등록할 키페어 생성
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
가상의 네트워크와 우분투 이미지를 등록하였으므로, 생성할 인스턴스에게 등록하기 위한 키페어를 생성합니다.
본 챕터에서는 mykey 라는 키페어를 생성하는 과정을 다룹니다.

 - 생성된 키페어 목록 조회
 - 키페어를 생성한 적이 없다면, 아무런 값도 반환되지 않습니다.

 .. code-block:: none

    openstack keypair list

|

 - 새로운 인스턴스에 등록할 신규 키페어를 생성하고, 이 키페어를 추가합니다.

 .. code-block:: none

    ssh-keygen -q -N "keypair name"
    openstack keypair create --public-ley ~/.ssh/id_rsa.pub <keypair name>

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ ssh-keygen -q -N "mykey"
    Enter file in which to save the key (/opt/stack/.ssh/id_rsa):

    stack@doa-wallaby-2:~/devstack$ openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
    +-------------+-------------------------------------------------+
    | Field       | Value                                           |
    +-------------+-------------------------------------------------+
    | created_at  | None                                            |
    | fingerprint | 1f:1c:c3:57:5e:94:5e:79:7e:50:21:9d:cf:df:78:0b |
    | id          | mykey                                           |
    | is_deleted  | None                                            |
    | name        | mykey                                           |
    | type        | ssh                                             |
    | user_id     | 04fa46bee5f04f548852a67db7f02371                |
    +-------------+-------------------------------------------------+

|

 - 생성한 키페어 목록을 조회합니다.

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack keypair list
    +-------+-------------------------------------------------+------+
    | Name  | Fingerprint                                     | Type |
    +-------+-------------------------------------------------+------+
    | mykey | 1f:1c:c3:57:5e:94:5e:79:7e:50:21:9d:cf:df:78:0b | ssh  |
    +-------+-------------------------------------------------+------+

|

보안 그룹 규칙 적용
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
위 글에서는 가상의 네트워크와 우분투 이미지, 생성할 인스턴스에게 등록하기 위한 키페어를 생성하였습니다.
이제 인스턴스를 생성하고, 원격 접속을 위한 SSH 프로토콜에 대한 Default 22번 포트를 특정 보안 그룹에 규칙을 추가합니다.
본 챕터에서는 SSH 접근을 허용하기 위한 정책을 추가하는 과정을 다룹니다.

|

 - 특정 보안 그룹의 규칙을 추가하기 위해서 보안 그룹 목록을 조회합니다.

 .. code-block:: none

    openstack security group list

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/.ssh$ openstack security group list
    +--------------------------------------+---------+------------------------+----------------------------------+------+
    | ID                                   | Name    | Description            | Project                          | Tags |
    +--------------------------------------+---------+------------------------+----------------------------------+------+
    | 11d4e899-1abd-41c8-944a-675c13cca53f | default | Default security group | 1790d56ebff849be81e7af3ac06a33c5 | []   |
    | 349628a1-7af1-4ec4-9766-b446a4123598 | default | Default security group | 81731826a82f437b8ac0f7a793916ce9 | []   |
    | 4424dbdc-5adc-4d99-96d7-cf64601e1f57 | default | Default security group | 940d7311e05d47e8bd397340aeb80c21 | []   |
    +--------------------------------------+---------+------------------------+----------------------------------+------+

|

 - 특정 보안 그룹에 대한 22번 포트를 허용합니다.

 .. code-block:: none

    openstack security group rule create --proto tcp --dst-port <port-number> <security-group ID>

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack security group rule create --proto tcp --dst-port 22 11d4e899-1abd-41c8-944a-675c13cca53f
    +-------------------------+--------------------------------------+
    | Field                   | Value                                |
    +-------------------------+--------------------------------------+
    | created_at              | 2021-10-17T14:08:24Z                 |
    | description             |                                      |
    | direction               | ingress                              |
    | ether_type              | IPv4                                 |
    | id                      | d5a418d7-bbed-4c8e-bb8e-0bc23b9cbe37 |
    | name                    | None                                 |
    | port_range_max          | 22                                   |
    | port_range_min          | 22                                   |
    | project_id              | 1790d56ebff849be81e7af3ac06a33c5     |
    | protocol                | tcp                                  |
    | remote_address_group_id | None                                 |
    | remote_group_id         | None                                 |
    | remote_ip_prefix        | 0.0.0.0/0                            |
    | revision_number         | 0                                    |
    | security_group_id       | 11d4e899-1abd-41c8-944a-675c13cca53f |
    | tags                    | []                                   |
    | updated_at              | 2021-10-17T14:08:24Z                 |
    +-------------------------+--------------------------------------+

|

인스턴스 생성
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
가상의 네트워크와 우분투 이미지, 키페어, 보안 규칙을 추가하였습니다.
이제 인스턴스를 생성할 준비가 되었습니다.
본 챕터에서는 인스턴스를 생성하는 과정을 다룹니다.


 - 신규 인스턴스 생성
 - 등록한 우분투 이미지에 할당 가능한 최소한의 Flavor 는 t1.small 입니다.

 .. code-block:: none

    openstack server create

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack server create --flavor m1.tiny \
    > --image ubuntu_20.04 \
    > --nic net-id=db6dafc1-8f4a-4079-a8d0-1c9c6e14d9c4 \
    > --security-group 11d4e899-1abd-41c8-944a-675c13cca53f \
    > --key-name mykey exam-instance

|

 - 생성한 인스턴스 목록을 조회합니다.
 - 생성된 인스턴스의 정보는 대시보드에서도 확인할 수 있습니다.

 .. code-block:: none

    openstack server list

 .. code-block:: none

    stack@doa-wallaby-2:~/devstack$ openstack server list
    +--------------------------------------+----------------+--------+----------------+--------------+----------+
    | ID                                   | Name           | Status | Networks       | Image        | Flavor   |
    +--------------------------------------+----------------+--------+----------------+--------------+----------+
    | 554d96dc-6542-46ee-9f2a-1bbb1800b5d0 | exam-instance  | ACTIVE | net1=10.8.0.77 | ubuntu_20.04 | m1.small |
    +--------------------------------------+----------------+--------+----------------+--------------+----------+

|

신규 인스턴스에 Floating IP 할당
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
축하합니다! 신규 인스턴스를 생성하였습니다.

새로운 인스턴스에 원격 접속을 위해 Floating IP를 할당하여야 합니다. 다른 네트워크 대역에서 인스턴스로 접근하기 위해서는 Floating IP를 설정하여야 합니다.
Floating IP는 기본적으로 퍼블릭 네트워크 대역의 호스트 아이피로 할당됩니다.

본 챕터에서는 인스턴스를 생성하는 과정을 다룹니다.


 - Floating ip 생성

 .. code-block:: none

    openstack floating ip create <public network>

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack floating ip create fccc03a4-16c7-425b-a163-0b58e7a01fdb
    +---------------------+--------------------------------------+
    | Field               | Value                                |
    +---------------------+--------------------------------------+
    | created_at          | 2021-10-17T14:23:01Z                 |
    | description         |                                      |
    | dns_domain          | None                                 |
    | dns_name            | None                                 |
    | fixed_ip_address    | None                                 |
    | floating_ip_address | 192.168.100.225                      |
    | floating_network_id | fccc03a4-16c7-425b-a163-0b58e7a01fdb |
    | id                  | 49455c7b-aacf-4da1-af81-549ced0bd4fb |
    | name                | 192.168.100.225                      |
    | port_details        | None                                 |
    | port_id             | None                                 |
    | project_id          | 1790d56ebff849be81e7af3ac06a33c5     |
    | qos_policy_id       | None                                 |
    | revision_number     | 0                                    |
    | router_id           | None                                 |
    | status              | DOWN                                 |
    | subnet_id           | None                                 |
    | tags                | []                                   |
    | updated_at          | 2021-10-17T14:23:01Z                 |
    +---------------------+--------------------------------------+

|

 - 생성한 floating ip 목록을 확인합니다.

 .. code-block:: none

    openstack floating ip list

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack floating ip list
    +--------------------------------------+---------------------+------------------+------+--------------------------------------+----------------------------------+
    | ID                                   | Floating IP Address | Fixed IP Address | Port | Floating Network                     | Project                          |
    +--------------------------------------+---------------------+------------------+------+--------------------------------------+----------------------------------+
    | 49455c7b-aacf-4da1-af81-549ced0bd4fb | 192.168.100.225     | None             | None | fccc03a4-16c7-425b-a163-0b58e7a01fdb | 1790d56ebff849be81e7af3ac06a33c5 |
    +--------------------------------------+---------------------+------------------+------+--------------------------------------+----------------------------------+

|

 - 신규 인스턴스에 Floating ip 를 할당합니다.

 .. code-block:: none

    openstack server add floating ip <instance ID> <floating-ip ID>

 .. code-block:: none

    stack@doa-wallaby-2:~/devstack$ openstack server add floating ip 554d96dc-6542-46ee-9f2a-1bbb1800b5d0 49455c7b-aacf-4da1-af81-549ced0bd4fb

|

 - 인스턴스에게 할당한 Floating ip 정보를 확인합니다.

 .. code-block:: none

    openstack floating ip list

 .. code-block:: none

    example)

    stack@doa-wallaby-2:~/devstack$ openstack floating ip list
    +--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
    | ID                                   | Floating IP Address | Fixed IP Address | Port                                 | Floating Network                     | Project                          |
    +--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+
    | 49455c7b-aacf-4da1-af81-549ced0bd4fb | 192.168.100.225     | 10.8.0.77        | a8168e50-65b5-4ce2-b6d8-0eed2f068759 | fccc03a4-16c7-425b-a163-0b58e7a01fdb | 1790d56ebff849be81e7af3ac06a33c5 |
    +--------------------------------------+---------------------+------------------+--------------------------------------+--------------------------------------+----------------------------------+

|

 - Floating ip를 할당한 인스턴스로 SSH 연결을 합니다.

 .. code-block:: none

    ssh -i <keypair-name> ubuntu@<instance floating-ip>

 .. code-block:: none

    stack@doa-wallaby-2:~/.ssh$ ssh -i mykey ubuntu@192.168.100.225
    The authenticity of host '192.168.100.225 (192.168.100.225)' can't be established.
    Are you sure you want to continue connecting (yes/no)? yes

|

