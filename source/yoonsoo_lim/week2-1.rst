
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

    * app.py한테 argv값을 전달하며