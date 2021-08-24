Openstack server list 명령어 동작 원리 파악
==========================================================

2주차 과제는 우리가 요청한 openstack server list의 값을 콘솔에 표시하기 까지 어떤 과정이 일어나는지
알아내는 과제였습니다. 워낙 방대한 파일이기 때문이기에 디버깅 하기에 다소 부족한 면이 있었습니다.

디버깅 툴로는 pycharm과 sublime text를 사용했습니다. pycharm의 경우 break point를 사용한
디버깅이 가능하다는 장점이 있었고 sublime text는 단어 검색에 빠르다는 장점이 있었습니다.

1.인자로 입력받은 server list를 어떻게 구별해내는가? 
______________________________________________________________________________________________

 1. 우선 가장 먼저 파일을 실행하는 python-openstackclient/openstackclient/shell.py에서 디버깅을 시작하였습니다.

    .. image:: ../images/week2-2_1.png
            :height: 300
            :width: 500

   위 사진과 같이 shell.py 는  main에서 OpenstackShell().run(argv) 를 실행하게 됩니다.
   이때 argv 로 리스트 형태로 ['server', 'list'] 값을 넘겨주게 됩니다.

 2. run 함수의 위치를 찾아가 보면 osc-lib/osc-lib/shell.py 에서 해당 함수를 찾을 수 있습니다.
    이 함수에서도 run 함수를 실행하기에 해당 함수로 진입해봅니다.

    .. image:: ../images/week2-2_2.png
            :height: 300
            :width: 700



 3. 2번의 run 함수는  python-openstackclient/venv/Lib/site-packages/cliff-3.8.0-py3.9.egg/cliff/app.py 에 위치하여 있습니다.
    여기서는 action 이라는 클래스를 디폴트 값으로 설정해주는 것 같습니다.


 4. 위의 run 함수를 실행하다보면 python-openstackclient/openstackclient/shell.py에 있는 initialize_app 이라는 함수를 실행하고 있음을
    볼 수 있습니다. app을 가동시킨다는 말로 보이는 해당 함수에 진입해보겠습니다.


 5. 해당 함수는 openstackclient/shell.py 의 _load_plugins 함수에서 필요한 plugin를 설정해주고 있습니다. 여기서 api, cmd_group 과 같은 값들이
    설정됨을 확인 할 수 있습니다. (연구 필요)
    

 6. 이렇게 initialize_app 함수가 끝나면 277번째 줄의 result = self.run_subcommand(remainder) 함수를 실행함을 볼 수 있습니다.
    이 함수를 따라가다보면 subcommand=self.command_manager.find_command(argv) 함수를 볼 수 있습니다.
    find_command 라는 걸 보면 우리가 원하는 명령어를 찾아주는 곳은 여기 인 듯 합니다. 이 함수의 반환값을 살펴보면
    cmd_factory, return_name, search_args를 반환하게 됩니다. 이 명령어를 찾는 과정은 아직 정확한 이해가 부족해 참고 사이트를
    첨부해두겠습니다.

    `참고자료 <https://epicarts.tistory.com/100>`_


2.server list 라는 명령어를 처리하는 파일은 무엇인가?
___________________________________________________________

 1. run 함수를 따라가다보면 openstackclient/compute/v2/server.py 의 ListServer를 실행하게 된다. 
    여기서 get_parser(prog_name) 를 통해 parser 정보를 설정하고 추가로 아래 코드들로 parser의 인자를 추가하고 있는듯 합니다.

 2. 이러한 설정을 거치고 다시 python-openstackclient/venv/Lib/site-packages/cliff-3.8.0-py3.9.egg/cliff/app.py 로 돌아와서 
    402번 줄의 cmd.run(parsed_args) 을 통해 해당 명령어를 처리하는 듯합니다. 

 3. command 명령어는 osc-lib/osc-lib/command/command.py 에 정의되어 있으며 39번째줄의 .run(parsed_args) 함수를 통하여
    python-openstackclient/venv/Lib/site-packages/cliff-3.8.0-py3.9.egg/cliff/display.py 의 run 함수를 실행시킨다.

 4. run함수에는 self.take_action(parsed_args)를 실행한다 여기서 실질적인 api 통신이 이루어지며 nova로 부터 결과를 받아온다. 


3. openstackcli는 어떻게 nova api 주소를 알아내나요?
___________________________________________________________

1. 연구중입니다.

4. nova의 어떤 API를 호출하여 결과를 받아오나요?
___________________________________________________________

 1. venv/Lib/site-packages/keystoneauth1-4.3.1-py3.9.egg/keystoneauth1/adapater.py 의 
    554번줄인 resp = super(LegacyjsonAdapter,self).request(args, kwargs) 함수를 통해 request한 값을 저장한다.

 

5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
_________________________________________________________________________________

 1. python-openstackclient/venv/Lib/site-packages/cliff-3.8.0-py3.9.egg/cliff/formatters/table.py
    의 emit_list 에서 출력 값을 형식을 생성함을 볼 수 있습니다.

 2. emit_list 내부 함수인 88번줄의 prettytable.prettyTable 과 96번줄의 self.add_rows 둘 중 하나의 함수로 예측하였는데 디버그 콘솔에서 확인하면
    prettytable.prettyTable 을 실행한 후 x 의 값은 여전히 비어있지만 add_rows 함수 수행후 우리가 흔히보는 출력형식이 나오므로
    add_rows 쪽이 좀 더 맞다고 판단하였습니다. 

 3. 이렇게 출력 형식이 지정되면 formatted= x.get.string() 과 stdout 함수를 통해 사용자의 콘솔에 출력함을 볼 수 있습니다.