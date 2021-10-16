3. cli로 floating ip 생성 후 인스턴스에 할당 / 해제 해보기
==========================================================


Floating IP 는 Fixed IP 처럼 자동으로 인스턴스에 default 로 할당되어 있지 않기 때문에 직접 인스턴스에 attach 해줘야 한다. 사용자들은 external network 로부터 인스턴스에 대한 연결성을 보장해주기 위해 cloud administrator 에 의해 정의된 다른 pool 부터 floating IP를 **grab** 해와야 한다.
(참고: `Blog <https://www.mirantis.com/blog/configuring-floating-ip-addresses-networking-openstack-public-private-clouds/>`_)

1. Floating IP 생성 후 인스턴스에 할당
---------------------------------------

솔직히 여기선 cloud administrator 가 정의한 다른 pool 이 뭔지 모르겠다. 하지만, **default 로 floating Ip address 는 public pool 로 부터 할당된다.**

다음과 같은 command로 floating address 를 현재 VM에 할당해보자.

.. code-block:: bash

   openstack floating ip create public

할당된 IP address 결과는 다음과 같다

.. image:: images/3-1.png

그런 다음과 같은 command 로 생성된 floating IPs 를 확인해보자

.. code-block:: bash

   openstack floating ip list

.. image:: images/3-2.png

그 결과 floating ip address 가 생성된 것을 확인할 수 있다.

그런 다음 command 로 생성된 floating ip address 를 instance 와 연결시켜주자.

.. code-block:: bash

   openstack server add floating ip "instance_name" "floating_ip_address"
   # openstack server add floating ip test_instance

`ERROR`_ 가 발생했다...

`공식문서 <https://docs.openstack.org/ocata/user-guide/cli-manage-ip-addresses.html>`_ 에서는 Private IP가 할당된 인스턴스에 floating ip를 할당해줘서 혹시나 해서 Private IP 로 설정해서 만든 인스턴스에 위와 같은 과정으로 floating ip address 를 해당 instance와 연결해줬더니 아무 이상 없이 floating IP가 할당되었다.

.. image:: images/3-3.png

위와 같이 floating ip가 인스턴스에 할당된 것을 볼 수 있다.

2. Floating IP 인스턴스에서 해제
---------------------------------------

인스턴스로부터 floating IP address 를 해제하기 위해 다음 command로 해제해주자.

.. code-block:: bash

   openstack server remove floating ip INSTANCE_NAME_OR_ID FLOATING_IP_ADDRESS

한 프로젝트로부터 floating IP address를 제거하기 위한 command 는 다음과 같다.

.. code-block:: bash

   openstack floating ip delete FLOATING_IP_ADDRESS

**제거된 IP 는 모든 프로젝트에서 이용 가능한 IP addresses 의 pool 로 리턴** 된다. 만약 **삭제한 IP address가 running instance에 할당되어 있으면 자동적으로 그 instance와 해제** 된다.

---------------------------

ERROR
------------------------

**생성한 floating ip address 를 instance 에 associate 해주는 과정에서 발생했다.**

.. warning::
    ResourceNotFound: 404: Client Error for url: http://211.37.148.128:9696/v2.0/floatingips/d47567b0-9a41-4aec-b733-d9b0d6a3cf26, External network **fe2d465c-4669-45de-8860-fa6373ef9ca2** is not reachable from subnet **bcec19b3-1711-4af9-80e2-b1b2fdd95457** . Therefore, cannot associate Port **7a8c850c-152b-434b-b900-2206375fc0a4** with a Floating IP.

- Instance Info
    - Image: cirros (default)
    - IP address: **Public(public=192.168.100.178, 2001:db8::2c7)**
    - Flavor: m1.tiny
- fe2d465c-4669-45de-8860-fa6373ef9ca2 : openstack public network ID
- bcec19b3-1711-4af9-80e2-b1b2fdd95457 : openstack public-subnet ID
- 7a8c850c-152b-434b-b900-2206375fc0a4 : ip_address=\'192.168.100.178\', subnet_id=\'bcec19b3-1711-4af9-80e2-b1b2fdd95457\'

`공식문서 <https://docs.openstack.org/ocata/user-guide/cli-manage-ip-addresses.html>`_ 에서는 Private IP가 할당된 인스턴스에 floating ip를 할당해줘서 혹시나 해서 Private IP 로 설정해서 만든 인스턴스에 위와 같은 과정으로 floating ip address 를 해당 instance와 연결해줬더니 아무 이상 없이 floating IP가 할당되었다.

.. image:: images/3-3.png

위와 같이 floating ip가 인스턴스에 할당된 것을 볼 수 있다.

Error 발생 이유
    public 네트워크로 인스턴스를 생성 시 생성된 인스턴스는 Clound VM 의 가상 네트워크를 사용하는 것이 아닌 실제로 존재하는 네트워크 주소를 가진 것이다. 현재 생성한 Floating IP 는 public pool 에서 가져온 것(실존 네트워크)이니 굳이 이 Floating IP 를 public 네트워크로 생성한 인스턴스에 할당해줄 필요가 없다. 서로 실존하는 네트워크가 겹쳐? 에러가 난 것이다.

해결 방안
    **public 이 아닌 private 네트워크로 생성한 인스턴스에 floating IP를 할당해주도록 하자!!!!**

참조
""""""

- `<https://www.mirantis.com/blog/configuring-floating-ip-addresses-networking-openstack-public-private-clouds/>`_
- `<https://docs.openstack.org/ocata/user-guide/cli-manage-ip-addresses.html>`_
- `<https://help.dreamhost.com/hc/en-us/articles/215912768-Managing-floating-IP-addresses-using-the-OpenStack-CLI>`_
