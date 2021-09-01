OpenStack 팀 3주차 진행사항 : openstack server list 에 field 추가하기
=======================================================================

3주차  과제에 대한 풀이 진행 사항입니다.

미션 1: openstack server list 의 기본 결과 필드에 "Project ID" 를 추가하기

미션 2: openstack server list 의 기본 결과 필드에 "Created At"를 추가하기

Mission 1 & 2 공동 해답
---------------------------

- server에 관한 결과 출력 사항을 보여주는 파일의 표 column에 변수를 추가합니다.

.. code-block:: python

        # python-openstackclient/compute/v2/server.py
        columns = (
            'ID',
            'Name',
            'Status',
            'Networks',
            'Image Name',
            'Flavor Name',
            'created', # 내 수정 영역
            "tenant_id", # 내 수정 영역
        )
        column_headers = (
            'ID',
            'Name',
            'Status',
            'Networks',
            'Image',
            'Flavor',
            'Created_At', # 내 수정 영역
            "Project_ID", # 내 수정 영역
        )

.. image:: https://miro.medium.com/max/700/1*ZCxhYS8N38r-fde87Hcvjg.png
   :width: 1000px
