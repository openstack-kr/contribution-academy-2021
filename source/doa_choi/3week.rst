============================================================
3주차 과제 - openstack server list 명령어에 field 추가하기
============================================================

00. openstack server list
-----------------------------------

openstack server list 명령어로 환경변수에 등록된 프로젝트의 모든 인스턴스를 조회해보면,
기본적으로 "ID", "Name", "Status", "Networks", "Image", "Flavor" 필드만 보이게 됩니다.
조회되는 필드에 "Project ID" 필드와 "Created At" 필드를 추가해주세요.

|

01. openstack server list 명령어 필드 추가
------------------------------------------------

**openstackclient\compute\v2\server.py ListServer.take_action메서드**

명령어가 출력되는 함수는 server.py 소스의 take_action 입니다.
해당 함수 내에서 하기 코드와 같이 수정하여 Project ID, Created At 을 추가하였습니다.


 .. code-block:: python
    :emphasize-lines: 8,9,18,19

                columns = (
                    'ID',
                    'Name',
                    'Status',
                    'Networks',
                    'Image Name',
                    'Flavor Name',
                    'Project ID',
                    'Created At'
                )
                column_headers = (
                    'ID',
                    'Name',
                    'Status',
                    'Networks',
                    'Image',
                    'Flavor',
                    'Project ID',
                    'Created At'
                )


|

수정 후, 출력되는 결과는 다음과 같습니다.

 .. image:: images/3week_01.png
