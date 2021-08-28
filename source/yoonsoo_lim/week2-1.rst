
2주차 과제 두번째 과제 - Openstack server list 명령어 동작 원리 파악
======================================================================



1) 첫 시작
----------------

1.1 python-openstackclient/openstackclient/shell.py
''''''''''''''''''''''''''''''''''''''''''''''''''''''''

    * 우리가 openstack server list 라는 명령어를 시작시키면 처음으로 실행되는 .py파일이다
    * 소스코드 143라인을 보면 sys.argv[1:]부분이 있는데 sys.argv를 출력해보면 아래와 같다.

    .. code-block:: python

        141    def main(argv=None):
        142        if argv is None:
        143            argv = sys.argv[1:]
        144
        145        return OpenStackShell().run(argv)

        ['C:…………/openstack/python-openstackclient/openstackclient/shell.py', 'server', 'list']

    * 위 처럼 우리가 server list라는 옵션을 주면 sys모듈을 이용하여 옵션을 받아온다.
    * 145L의 run은 osc_lib/shell.py의 run을 실행시킨다.


|

1.2 osc_lib/shell.py
'''''''''''''''''''''''''

    * 133L의 super(OpenStackShell, self).run(argv)를 실행시키면 우리가 원하는 server list의 결과값이 나오는걸 확인할 수 있다.

    .. code-block:: python

        129    def run(self, argv):
        130        ret_val = 1
        131        self.command_options = argv
        132        try:
        133            ret_val = super(OpenStackShell, self).run(argv)
        134            return ret_val


    * run의 클래스를 확인 해보자

    .. code-block:: python

        72     class OpenStackShell(app.App):

    * run의 클래스는 app.App을 상속받아서 사용한다. 맨위 import를 확인해보면 cliff의 app인것을 확인할 수 있다.

    .. code-block:: python

        24    from cliff import app


|

1.3 cliff/app.py
''''''''''''''''''''''

    * app.py의 run()한테 argv값(server list)을 전달한다.
    * 257L은 제일 중요한 initialize작업을 해준다. (initialize이란 cli환경에서 어떤 기능들을
      제공할지 정보들을 로딩에서 환경을 만들어주는 작업이다.)

    .. code-block:: python

        File: cliff/app.py

        257 self.initialize_app(remainder)

    * 함수 안으로 들어가보면 openstackclient/shell.py의 initialize_app을 호출하는데
      이 함수는 또 osc_lib/shell.py의 initialize_app을 또 호출한다.

    .. code-block:: python

        File: openstackclient/shell.py

        130 def initialize_app(self, argv):
        131     super(OpenStackShell, self).initialize_app(argv)


        File: osc_lib/shell.py

        388 def initialize_app(self, argv):


    * 여기서 제일 중요한것은 442,444L이다.
    * self._load_plugins()는 cmd_group에 openstack.compute.v2를 만들어낸다.

    .. code-block:: python

        File: osc_lib/shell.py

        442 self._load_plugins()

    * 아래는 위에 ._load_plugins를 타고 들어간곳이다.
    * 71L에 clientmanager.PLUGIN_MODULES가 뭐하는 함수인지는 모르겠지만,
      102L에 add_command_group부터 계속 타고 들어가면 commandmanager.py의
      load_commands가 나온다.

    .. code-block:: python

        File: openstackclient/shell.py

        65 def _load_plugins(self):
        71    for mod in clientmanager.PLUGIN_MODULES:

        File: cliff/commandmanager.py

        70 def load_commands(self, namespace)
        73     for ep in stevedore.ExtensionManager(namespace):
        78          self.commands[cmd_name] = ep.entry_point

    * 73L을 보면 stevedore라는게 있는게 이것은 Python 애플리케이션용
      동적으로 플러그인 관리를 해주는 모듈이라고 한다.

    |

    * 결론! self._load_plugins()가 하는일!
      clientmanager의 PLUGIN_MODULES가 어떻게 동작하는지는 모르겠지만,
      그걸 이용해서 stevedore라는 모듈을 통해 setup.cfg에 정의된 정보들을
      읽어오는구나를 알수있다!

    |

    * 이젠 self._load_commands()를 확인해보자.
      계속 타고 들어가면 아까 cliff/commandmanager.py로 똑같이 들어온다.

    .. code-block:: python

        70 def load_commands
        73     for ep in stevedore.ExtensionManager(namespace):

    * 이번엔 namespace가 openstack.common으로 바뀌고 함수를 실행한다.

    |

    * 결국 stevedore를 이용해 우리가 입력하는 옵션(ex:server list)들을 긁어오기 위해
      필요한 모듈을 전부 로딩해놓는것을 확인할 수 있다.


    * 그리고 277L에 run_subcommand를 실행시키는데 저것이 무엇인지 한번 보자

    .. code-block:: python

        279    result = self.run_subcommand(remainder)

    * run_subcommand함수의 365L을 보면 find_command라는 함수가 있는데 뭔가
      명령어를 실행시키는 파일을 찾는거 같다.

    .. code-block:: python

        365    subcommand = self.command_manager.find_command(argv)


1.4 cliff/commandmanager.py
'''''''''''''''''''''''''''''''

    * find_command함수의 114~115L의 self.commands를 출력을 해보면 여러 명령어들이 딕셔너리 형태로
      들어가 있다.
    * 그 중에 name(value: server list)의 값이 명령어 딕셔너리에 있으면 그것을 found에 넣는다.

    .. code-block:: python

        97    def find_command(self, argv):
        114     if name in self.commands:
        115         found = name


    * self.commands의 키값중에 found(server list)에 해당하는 value값을 cmd_ep에 넣는데
      그 값을 확인 해보면 server list라는 옵션은 compute/v2/server.py의 ListServer라는
      함수가 처리하는거 같다!

    .. code-block:: python

        123    cmd_ep = self.commands[found]

    .. code-block:: python

        >>>  print(cmd_ep)
        EntryPoint(name='server_list', value='openstackclient.compute.v2.server:ListServer', group='openstack.compute.v2')

    * 그리고 131L을 통해 Entrypoint를 load해서 cmd_factory에 인스턴스화 시킨다.

    .. code-block:: python

        File: cliff/commandmanager.py

        131 cmd_factory = cmd_ep.load()

    * 다시 app.py로 넘어와 384L에서 변수들에게 값을 전달하고 cmd_factory는
      cmd에다가 다시 인스턴스화를 한다.

    |

    * 마지막으로 402L을 통해 server list의 결과값을 얻는다.
      이것도 안으로 계속 들어가보자

    .. code-block:: python

        402 result = cmd.run(parsed_args)

    * 그럼 중간 cliff/display.py의 run을 실행시킨다.
    * 여기서 115L에 take_action을 호출하는데 이것은
      openstackclient.compute.v2.server.ListServer의 take_action이다.

    .. code-block:: python

        File: cliff/display.py
        112 def run(self, parsed_args):
        115     column_names, data = self.take_action(parsed_args)

            >>> self.take_action
            <bound method ListServer.take_action of <openstackclient.compute.v2.server.ListServer object at 0x00000260BA41BF70>>

    * 이 함수를 통해 입력한 server list옵션에 해당하는 결과값을 return받아온다.