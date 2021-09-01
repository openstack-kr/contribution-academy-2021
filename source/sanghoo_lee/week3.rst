3주차 과제 : openstack server list 에 field 추가하기
==========================================================


openstack server list 를 하면 기본적으로 "ID", "Name", "Status", "Networks" ,"Image", "Flavor" 필드만 보입니다.

.. code-block::


    +--------------------------------------+---------------+--------+---------------------------------------------------------------------------------+--------------------------+-----------+
    | ID                                   | Name          | Status | Networks                                                                        | Image                    | Flavor    |
    +--------------------------------------+---------------+--------+---------------------------------------------------------------------------------+--------------------------+-----------+
    | ae5fb280-5115-4905-80b3-64e2c8f35f5b | demo-instance | ACTIVE | private=10.0.0.17, fd04:f6e0:afc3:0:f816:3eff:fe89:f64c; shared=192.168.233.216 | N/A (booted from volume) | cirros256 |
    +--------------------------------------+---------------+--------+---------------------------------------------------------------------------------+--------------------------+-----------+


중요 포인트는 이와 같다고 생각했다.

1. "ID", "Name", "Status", .. 총 6개의 data 만 받아오는 걸까?

2. 데이터를 받아서 파싱해주는 위치는 어디일까?


미션 : openstack server list 의 기본 결과 필드에 "Project ID", "Created At" 를 추가하기
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""

1. data 를 받아오는 곳은 API 를 호출하는 곳이다.

- week2-2.rst 하단 참조
self.api.client.get(url) 는 keystoneauth1/adapter.py 를 호출한다.

.. code-block:: python

   # keystoneauth1/adapter.py
        def get(self, url, **kwargs):
        return self.request(url, 'GET', **kwargs)

resp = 200
body = instance 정보

가 반환된다.

반환받은 instance 의 정보를 확인해보면,


.. code-block::

   0 = {dict: 30} {'id': 'ae5fb280-5115-4905-80b3-64e2c8f35f5b', 'name': 'demo-instance', 'status': 'ACTIVE', 'tenant_id': '02f735dca6e04eacb54c914c51729d80', 'user_id': 'd0ea89b5858e49d88961d1d0e91a641c', 'metadata': {}, 'hostId': '06a4a6ba5ada1a97a5ae0828057f8267f77d0d9a93bcdce137012c04', 'image': '', 'flavor': {'id': 'c1', 'links': [{'rel': 'bookmark', 'href': 'http://211.37.148.101/compute/flavors/c1'}]}, 'created': '2021-08-30T05:37:52Z', 'updated': '2021-08-30T05:38:22Z', 'addresses': {'private': [{'version': 6, 'addr': 'fd04:f6e0:afc3:0:f816:3eff:fe89:f64c', 'OS-EXT-IPS:type': 'fixed', 'OS-EXT-IPS-MAC:mac_addr': 'fa:16:3e:89:f6:4c'}, {'version': 4, 'addr': '10.0.0.17', 'OS-EXT-IPS:type': 'fixed', 'OS-EXT-IPS-MAC:mac_addr': 'fa:16:3e:89:f6:4c'}], 'shared': [{'version': 4, 'addr': '192.168.233.216', 'OS-EXT-IPS:type': 'fixed', 'OS-EXT-IPS-MAC:mac_addr': 'fa:16:3e:4f:a2:97'}]}, 'accessIPv4': '', 'accessIPv6': '', 'links': [{'rel': 'self', 'href': 'http://211.37.148.101/compute/v2.1/servers/ae5fb280-...
 'id' = {str} 'ae5fb280-5115-4905-80b3-64e2c8f35f5b'
 'name' = {str} 'demo-instance'
 'status' = {str} 'ACTIVE'
 'tenant_id' = {str} '02f735dca6e04eacb54c914c51729d80'
 'user_id' = {str} 'd0ea89b5858e49d88961d1d0e91a641c'
 'metadata' = {dict: 0} {}
 'hostId' = {str} '06a4a6ba5ada1a97a5ae0828057f8267f77d0d9a93bcdce137012c04'
 'image' = {str} ''
 'flavor' = {dict: 2} {'id': 'c1', 'links': [{'rel': 'bookmark', 'href': 'http://211.37.148.101/compute/flavors/c1'}]}
 'created' = {str} '2021-08-30T05:37:52Z'
 'updated' = {str} '2021-08-30T05:38:22Z'
 'addresses' = {dict: 2} {'private': [{'version': 6, 'addr': 'fd04:f6e0:afc3:0:f816:3eff:fe89:f64c', 'OS-EXT-IPS:type': 'fixed', 'OS-EXT-IPS-MAC:mac_addr': 'fa:16:3e:89:f6:4c'}, {'version': 4, 'addr': '10.0.0.17', 'OS-EXT-IPS:type': 'fixed', 'OS-EXT-IPS-MAC:mac_addr': 'fa:16:3e:89:f6:4c'}], 'shared': [{'version': 4, 'addr': '192.168.233.216', 'OS-EXT-IPS:type': 'fixed', 'OS-EXT-IPS-MAC:mac_addr': 'fa:16:3e:4f:a2:97'}]}
 'accessIPv4' = {str} ''
 'accessIPv6' = {str} ''
 'links' = {list: 2} [{'rel': 'self', 'href': 'http://211.37.148.101/compute/v2.1/servers/ae5fb280-5115-4905-80b3-64e2c8f35f5b'}, {'rel': 'bookmark', 'href': 'http://211.37.148.101/compute/servers/ae5fb280-5115-4905-80b3-64e2c8f35f5b'}]
 'OS-DCF:diskConfig' = {str} 'AUTO'
 'progress' = {int} 0
 'OS-EXT-AZ:availability_zone' = {str} 'nova'
 'config_drive' = {str} ''
 'key_name' = {NoneType} None
 'OS-SRV-USG:launched_at' = {str} '2021-08-30T05:38:21.000000'
 'OS-SRV-USG:terminated_at' = {NoneType} None
 'OS-EXT-SRV-ATTR:host' = {str} 'r-hoo-wallaby'
 'OS-EXT-SRV-ATTR:instance_name' = {str} 'instance-00000001'
 'OS-EXT-SRV-ATTR:hypervisor_hostname' = {str} 'r-hoo-wallaby'
 'OS-EXT-STS:task_state' = {NoneType} None
 'OS-EXT-STS:vm_state' = {str} 'active'
 'OS-EXT-STS:power_state' = {int} 1
 'os-extended-volumes:volumes_attached' = {list: 1} [{'id': '0ab259fa-d64e-4898-9845-1798c8e0b0c5'}]
 'security_groups' = {list: 2} [{'name': 'default'}, {'name': 'default'}]
 __len__ = {int} 30
....


이와 같이 모든 정보가 전달되고 있음을 알 수 있다.



2. 데이터를 받아서 파싱해주는 위치는 어디일까?


data 를 반환받은 위치로 다시 돌아와보니,
반환 받기 전부터 column_header 값이 정의 되어 있음을 확인 할 수 있었다.

column_headers = {tuple: 6}('ID', 'Name', 'Status', 'Networks', 'Image', 'Flavor')

.. code-block:: python

    #  compute/v2/server.py
    column_headers = tuple(column_headers)      # 여기서 선언된것 같다.
            columns = tuple(columns)

        if parsed_args.marker:
            # Check if both "--marker" and "--deleted" are used.
            # In that scenario a lookup is not needed as the marker
            # needs to be an ID, because find_resource does not
            # handle deleted resources
            if parsed_args.deleted:
                marker_id = parsed_args.marker
            else:
                marker_id = utils.find_resource(compute_client.servers,
                                                parsed_args.marker).id

        data = compute_client.servers.list(search_opts=search_opts,
                                           marker=marker_id,
                                           limit=parsed_args.limit)




코드를 역추적해보니 같은 소스파일에 지정해주는 위치가 있었다.

.. code-block::

               else:
                columns = (
                    'ID',
                    'Name',
                    'Status',
                    'Networks',
                    'Image Name',
                    'Flavor Name',
                    'tenant_id',
                    'created',
                )
            column_headers = (
                'ID',
                'Name',
                'Status',
                'Networks',
                'Image',
                'Flavor',
                'Project ID',
                'Created At',
            )

따라서 data 에서 받은 인자값을 참고하여 해당 tuple에 tenant_id, created 을 추가하였다.

+) created_at 이라는 값이 data 에 없어서 출력이 되지 않는 issue 가 있었다.
같은 팀원인 이재용님의 PR 을 참조하겠다.
https://review.opendev.org/c/openstack/python-openstackclient/+/806464
