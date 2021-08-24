
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
    * 그리고 279L에 run_subcommand를 실행시키는데 저것이 무엇인지 한번 보자

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

