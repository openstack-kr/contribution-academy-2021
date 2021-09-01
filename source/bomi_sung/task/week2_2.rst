============================================
openstack server list 명령어 동작 원리 파악
============================================

-----
목표
-----
  - openstack server list 는 환경변수에 등록된 프로젝트의 모든 인스턴스를 출력하는 명령어입니다.
  - python-openstackcli 의 코드 내부에서 이 명령어를 처리하기 위한 절차를 분석 후 자유롭게 보고서로 작성해주세요.

-----

----------------------------------------------------
인자로 입력받은 server list 를 어떻게 구별해내는가?
----------------------------------------------------

먼저, Pycharm 의 python-openstackclient 의 프로젝트에서 cli 명령어 실행 시 시작 지점이 되는 shell.py 파일의 main 에 break point 를 걸고 분석해보았다.

  .. image:: ../images/week2/image_13.png

입력한 인자 값이 ``argv[0] = server, argv[1] = list`` 로 들어오는 것을 확인할 수 있고 openstackclient/shell.py 의 main 에서 ``OpenStackShell().run()`` 을
실행하게 되면 상속 관계에 따라 osc_lib/shell.py 의 OpenStackShell 클래스의 run() 함수가 호출되고 함수 내부에서 ``super(OpenStackShell, self).run(argv)`` 명령어가 호출되면
cliff/app.py 의 App class 에서 각 명령어들에 대한 초기화 작업을 수행한다. (OpenStackShell 클래스는 3단계의 상속 관계를 가지고 있음을 알 수 있다)

초기화 과정에서 openstackclient/shell.py 의 ``load_plugins()`` 함수 내에서 모듈을 하나씩 불러와서 cmd_group 문자열을 생성한 후 ``command_manager.add_command_group(cmd_group)`` 을 호출한다.

  .. image:: ../images/week2/image_14.png

한 줄씩 따라 들어가면 ``cliff/commandmanager.py`` 파일의 CommandManager class 내부로 들어가게 되는데
``load_commands()`` 함수를 살펴보면 stevedore.ExtensionManager 에서 값을 가져오는 걸을 볼 수 있다.
stevedore 는 openstack 에서 만든 컴포넌트인데 여러 개의 plugin 들을 동적으로 로딩할 수 있게 해주는 라이브러리로
openstack 의 CLI 명령어 정보를 가져오기 위해 stevedore 를 사용해 모듈을 로딩해오는 것이다.

(참고 : https://docs.openstack.org/stevedore/latest/ )

  .. image:: ../images/week2/image_15.png

코드를 살펴보면 ``load_commands()`` 함수에서 ep.name 에 담겨있는 String 의 언더바(_) 를 하이픈(-) 으로 치환하여
cmd_name 변수에 담아주고 있는 것을 볼 수 있다.

  .. image:: ../images/week2/image_16.png

이 값은 entry_points.txt 파일에서 확인할 수 있는데 파일에 있는 명령어 리스트를 읽어와서 commands 리스트에 담아주는 것을 추측해볼 수 있다.

  .. image:: ../images/week2/image_17.png

이후 ``find_command()`` 함수 내부에서 인자로 입력받은 ``server list`` 명령어가 commands 리스트에 있는지 찾아주는 과정을 거치게 된다.

  .. image:: ../images/week2/image_18.png

전체적인 흐름을 정리해보면 stevedore 를 사용해 모듈을 로딩하여 CommandManager 가 해당 정보들을 가지고 있다가 인자로 넘어온 값이 command list 에 있으면
해당 명령어의 entry point 를 반환해주고 각 명령어들이 공통적으로 가지고 있는 take_actions() 함수를 호출하여 명령어를 실행하게 된다.

-----

-----------------------------------------------------
server list  라는 명령어를 처리하는 파일은 무엇인가?
-----------------------------------------------------

아래와 같이 ``openstackclient.compute.v2.server.py`` 파일에서 처리되는 것을 확인할 수 있다.

  .. image:: ../images/week2/image_19.png

-----

------------------------------------------------------------
결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
------------------------------------------------------------

  .. image:: ../images/week2/image_20.png

server list 명령어 입력 시 server 관련 정보를 위와 같이 출력해주는 것에 힌트를 얻어 테이블의 컬럼이 포함하는
``Status`` 문자열을 ``grep –r “Status” ./*`` 명령어를 사용해서 검색하였다.

  .. image:: ../images/week2/image_21.png

server 관련 파일로 보이는 ``./compute/v2/server.py`` 파일을 열어서 ``Status`` 문자열을 검색해보며 범위를 좁혀나갔다.
해당 파일에는 ``Status`` 문자열이 다수 포함되어 있는데 그 중 ListServer class 의  ``take_action()`` 함수에서 아래와 같이 table을 만들어 반환해주는 것을 확인할 수 있었다.

  .. image:: ../images/week2/image_22.png

찾은 곳이 server list 의 정보를 출력해주는 부분이 맞는지 확인하기 위해 print(table)를 추가한 후 명령어를 실행해보았더니 테이블의 컬럼명이 동일하게 출력되는 것을 확인할 수 있었다.

  .. image:: ../images/week2/image_23.png

검색을 통해 Python에서 openstack 에서와 같이 데이터를 테이블 형식으로 출력해주는 모듈에는 ``PrettyTable, termtable, texttable`` 등이 있는 것을 알아냈고
grep 으로 PreetyTable을 검색했더니 아래와 같은 결과 화면을 볼 수 있었다.

  .. image:: ../images/week2/image_24.png

이를 통해 openstack 에서는 테이블 출력 시 PreetyTable 모듈을 사용하는 것을 확인할 수 있는데 위에서 찾은 바와 같이 ListServer class 에서 server list 정보를
table 로 반환해주고 이를 PreetyTable 모듈을 사용하여 테이블로 출력해줄 것임을 알 수 있다.

-----

----------------------------------------------------------------------------
nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )
----------------------------------------------------------------------------

`<Note>`
  Nova는 오픈 스택 프로젝트 중 하나이며, 컴퓨트 인스턴스(가상 서버) 프로비져닝 서비스를 제공해준다. (참고 : https://docs.openstack.org/nova/latest/)

앞서서 server list 결과를 출력하기 위해 관련 정보를 compute/v2/server.py 의 ListServer class 의 ``take_action()`` 에서 가져오는 것을 확인했었다.
``take_action()`` 함수 내부를 좀 더 자세히 살펴보면 아래와 같이 data 변수에서 novaclient/v2/client.py 파일의 Client class 내부로 들어가고
초기화 과정에서 ``self.servers = servers.ServerManager(self)`` 를 실행하는 것을 볼 수 있었다.

  .. image:: ../images/week2/image_26.png

novaclinet/v2/servers.py 파일의 ServerManager class 내부로 따라 들어가면 list() 함수를 볼 수 있는데 ``/servers/detail`` URI 를 호출해서 server list 정보를 가져오는 것을 확인할 수 있다.

  .. image:: ../images/week2/image_27.png

