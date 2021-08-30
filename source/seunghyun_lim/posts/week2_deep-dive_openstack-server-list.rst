2주차 과제 - openstack server list 명령어 동작 원리 파악
===================================

openstack server list 는 환경변수에 등록된 프로젝트의 모든 인스턴스를 출력하는 명령어입니다.

python-openstackcli 의 코드 내부에서 이 명령어를 처리하기 위한 절차를 분석 후 자유롭게 보고서로 작성해주세요.

주요 분석 포인트
-----------------------------
#. 인자로 입력받은 server list 를 어떻게 구별해내는가?
    #. Cliff.app.App.run
        * Cliff.app.App.run 함수에서 parser로 Openstackcli에 필요한 환경설정 변수와 명령어(remainder)를 구분한다.
            * `def run(self, argv): self.options, remainder = self.parser.parse_known_args(argv)`
        * 명령어 부분은 `Cliff.app.App.run_subcommand(self, argv)`를 호출하여 처리한다.
#. server list  라는 명령어를 처리하는 파일은 무엇인가?
    #. python-openstackclient.openstackclient.compute.v2.server.py의 ListServer 클래스
    #. Cliff.app.App.run_subcommand(self, argv)
        * `subcommand = self.command_manager.find_command(argv)`로 command_manager가 argv (server list)를 처리하는 함수를 리턴으로 돌려준다.
    #. Cliff.commandmanager.CommandManager.find_command(self, argv)
        * 명령어는 "server list"처러 String으로 만들어져 found에 할당된다.
        * 그 후 `cmd_ep = self.commands[found]`를 통해 self.commands:dictionary에 있는 server list를 처리할 수 있는 클래스를 리턴한다.
    #. 추가로 self.commands는 CommandManager 클래스가 선언 될 때 load_commands를 통해 stevedore.ExtensionManager에서 명령어와 그에 맞는 클래스를 받아온다.
    #. 받아온 명령어를 처리하는 클래스는 #. python-openstackclient.openstackclient.compute.v2.server.py의 ListServer 클래스이다.
#. openstackcli 는 어떻게 nova api 주소를 알아내나요?
    * app.client에서 미리 계산된 주소를 담고 있다.
#. nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )
    * novaclient/servers.py "/servers/detail"
#. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
    * listener가 parser의 내용와 nova api를 완료한 결과를 받아와 cliff/formatter/table.py로 전달, table의 emit_list에서 실행 결과를 pretty_table의 get_string을 이용하여 그려준다.


전체 과정 분석
------------------
1. argv : openstack server list
2. OpenStackShell().run(argv)

.. code-block:: python

    def main(argv=None):
        if argv is None:
            argv = sys.argv[1:]

        return OpenStackShell().run(argv)
|
    2-1. python-openstackclient.openstackclient::OpenStackShell()
        - var command_manager = osc_lib.CommandManager('openstack.cli')
        - osc_cli.shell.OpenstackShell을 상속함
            - cliff.app.App을 상속함
    2-2. osc_lib.shell.run(argv)
.. code-block:: python

    class OpenStackShell(app.App):
        def run(self, argv):
            ret_val = 1
            self.command_options = argv
            ret_val = super(OpenStackShell, self).run(argv)
            return ret_val

|

