==========================================================
2week - cli와 친해지기
==========================================================

문제 1) cirros image로 인스턴스 생성을 cli로 해보기
----------------------------------------------------------
-	openstack server create


1-1) openstackserver create를 검색해서 방법을 찾아봅니다.

https://docs.openstack.org/mitaka/ko_KR/install-guide-rdo/launch-instance-selfservice.html

1-2) flavor list 확인

.. code-block:: python

	openstack flavor list

1-3) image list 확인

.. code-block:: python

	openstack image list

1-4) network list 확인

.. code-block:: python

	openstack network list

1-5) openstack server create로 합치기

.. code-block:: bash
	
	openstack server create –flavor m1.small –image ubuntu-pwd –nic net-id=ca77af0e-a022-4208-84a1-58451dabb6d4 ubuntu_test


문제 2) ubuntu 이미지를 받고, root password를 설정한 다음 cli로 이미지 등록한 후 인스턴스 생성하고 접속까지 하기
------------------------------------------------------------------------------------------------------------------------
-	ubuntu image: https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img

-	만약 root password 설정을 못하겠다면, 인스턴스 생성 후 로그인 창 까지만 떠도 OK

2-1) wget 명령어로 이미지를 가져옵니다.

.. code-block:: python

	wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img

2-2) ubuntu cloud image root password 를 검색해서 방법을 찾아봅니다.

https://askubuntu.com/questions/451673/default-username-password-for-ubuntu-cloud-image

2-3) 다음 과정을 진행합니다. 패스워드를 openstack으로 설정했습니다.

.. code-block:: python

	sudo apt install libguestfs-tools
	sudo virt-customize -a bionic-server-cloudimg-amd64.img --root-password password:openstack

2-4) password를 설정한 이미지를 생성하기 위해 image create 방법을 찾아봅니다.

https://docs.openstack.org/newton/ko_KR/install-guide-rdo/glance-verify.html

2-5 이미지를 생성합니다.

.. code-block:: python
	
	openstack image create "ubuntu" --file focal-server-cloudimg-amd64.img --disk-format qcow2 --container-format bare --public

2-6 인스턴스를 생성합니다. 인스턴스 생성은 1번 과제를 참고하면 됩니다.

.. code-block:: python
	
	openstack server create --flavor m1.small --image ubuntu --nic net-id=ca77af0e-a022-4208-84a1-58451dabb6d4  ubuntu_pwd




문제 3) cli로 floating ip 생성 후 인스턴스에 할당/해제 해보기
---------------------------------------------------------------------------------------

3-1) openstack floating ip create를 검색해봅니다.

https://docs.openstack.org/ocata/user-guide/cli-manage-ip-addresses.html

3-2) private network를 가지는 인스턴스를 생성합니다.

.. code-block:: python
	
	openstack server create --flavor m1.small --image ubuntu --network=private ubuntu

3-3) 남는 floating ip를 할당합니다.

.. code-block:: python
	
	openstack floating ip create public


3-4) server에 위에서 할당받은 floating ip를 add 해줍니다.

.. code-block:: python
	
	openstack server add floating ip ubuntu 192.168.100.64

3-5) remove 명령을 통해 할당된 floating ip를 해제할 수 있습니다.

.. code-block:: python
	
	openstack server remove floating ip ubunu 192.168.100.64



	
문제 4) 10.8.0.0/24 네트워크를 만들고 public network와 연결하는 과정을 cli로 해보기(optional)
-----------------------------------------------------------------------------------------------------------------

4-1) openstack network create를 검색해서 네트워크를 만드는 과정을 검색해봅니다.

https://docs.openstack.org/ocata/user-guide/cli-create-and-manage-networks.html

4-2) network를 만듭니다.

.. code-block:: python
	
	openstack network create net1

4-3) subnet을 만듭니다.

.. code-block:: bash
	
	openstack subnet create subnet1 –network net1 –subnet-range 10.8.0.0/24

10.8.0.0/24의 네트워크를 만들었습니다.

10.8.0.0/24의 네트워크와 public 네트워크를 연결하기 위해 라우터를 만듭니다.

4-4) router를 만듭니다.

.. code-block:: python
	
	openstack router create router1

4-5) router에 subnet들을 add합니다.

가장 먼저 openstack router list / openstack subnet list를 확인합니다.

openstack router add subnet [router] [subnet] 명령어를 사용해서 router에 subnet들을 연결합니다.

네트워크 토폴로지에서 10.8.0.0 subnet과 public이 연결된 걸 확인할 수 있습니다.

.. image:: ../images/cli_1.jpg
	:height: 500
	:width: 500
	:alt: flavor list
	
	
	
이번 과제는 docs에서 내용이 많이 나온 것 같습니다!