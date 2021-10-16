2week - openstack server list 명령어 동작 원리 파악
======================================================================

1. 인자로 입력받은 server list를 어떻게 구별해내는가?
**********************************************************************

'openstack server list' 인자 값을 입력 시 다음의 테이블을 보여준다.

.. code-block:: python

   openstack server list

+--------------------------------------+----------------+--------+---------------------------------------------------------+--------------------------+----------+
| ID                                   | Name           | Status | Networks                                                | Image                    | Flavor   |
+--------------------------------------+----------------+--------+---------------------------------------------------------+--------------------------+----------+
| 9977e331-2ad9-4e1c-9f1b-5ed13b3af425 | ubunturootpass | ACTIVE | shared=192.168.233.66                                   | ubuntu-18.04             | ds512M   |
| 548c69ac-9381-43cb-9772-4019fed865d5 | jiwonmyserver  | ACTIVE | private=10.0.0.42, fdb3:1a14:fc41:0:f816:3eff:fe28:c51c | cirros-0.5.2-x86_64-disk | m1.small |
+--------------------------------------+----------------+--------+---------------------------------------------------------+--------------------------+----------+

분석하기_1단계 OpenStack 명령어의 실행 위치 알아내기
-----------------------------------------------------------------------

.. code-block:: html
   :emphasize-lines: 2

   stack@yjw-wallaby:~/devstack$ which openstack
   /usr/local/bin/openstack

.. code-block:: python

   view /usr/local/bin/openstack

   #!/usr/bin/python3.6
   # -*- coding: utf-8 -*-
   import re
   import sys
   from openstackclient.shell import main
   if __name__ == '__main__':
      sys.argv[0] = re.sub(r'(-script\.pyw|\.exe)?$', '', sys.argv[0])
      sys.exit(main())

**#!** 은 openstack 스크립트에서 실행시켜줄 프로그램의 경로를 지정하는 역할이다.

-> /usr/bin/python3.6 프롬프트을 사용한다는 것을 알 수 있다.

핵심 함수의 파일 위치 알아내기
-----------------------------------------------------------------------

**main** () 함수를 실행하는 것을 알 수 있다. **main** 은 **openstackclient.shell** 이므로
해당 프롬프트가 사용하는 패키지의 **openstackclient** 폴더에서 **shell.py** 파일을 main 변수로 정의한 것을 알 수 있다.

**/usr/bin/python3.6** 실행한 후 아래의 예시처럼 위치를 알아낼 수 있다.

.. image:: images/picture_2.png

openstack server list 처리 과정
-----------------------------------------------------------------------

따라서 openstack server list 명령어를 입력하면 openstackclient/shell.py 파일이 실행된다.

.. code-block:: python

   view /usr/local/lib/python3.6/dist-packages/openstackclient/shell.py

**main** 함수로 아무런 인자를 받지 않았으면 **argv** 에 server list 값을 할당한다.

OpenStackShell 클래스의 run 함수로 server list 값을 넘겨주는 것을 볼 수 있다.

**shell.py**

.. code-block:: python

   """Command-line interface to the OpenStack APIs"""

   import sys

   from osc_lib.api import auth
   from osc_lib.command import commandmanager
   from osc_lib import shell

   import openstackclient
   from openstackclient.common import clientmanager


   DEFAULT_DOMAIN = 'default'


   class OpenStackShell(shell.OpenStackShell):

       def __init__(self):

           super(OpenStackShell, self).__init__(
               description=__doc__.strip(),
               version=openstackclient.__version__,
               command_manager=commandmanager.CommandManager('openstack.cli'),
               deferred_help=True)

           self.api_version = {}

           # Assume TLS host certificate verification is enabled
           self.verify = True

       def build_option_parser(self, description, version):
           parser = super(OpenStackShell, self).build_option_parser(
               description,
               version)
           parser = clientmanager.build_plugin_option_parser(parser)
           parser = auth.build_auth_plugins_option_parser(parser)
           return parser

    ...


      def main(argv=None):
          if argv is None:
              argv = sys.argv[1:]

          return OpenStackShell().run(argv)


      if __name__ == "__main__":
          sys.exit(main())

OpemStackShell은 shell로 부터 OpenStackShell 클래스를 상속받으며

.. code-block:: python

   class OpenStackShell(shell.OpenStackShell):

       def __init__(self):

           super(OpenStackShell, self).__init__(
               description=__doc__.strip(),
               version=openstackclient.__version__,
               command_manager=commandmanager.CommandManager('openstack.cli'),
               deferred_help=True)

**shell** 은 **osc_lib** 의 shell 파일이다.

.. code-block:: python

   from osc_lib import shell

**osc_lib/shell.py**

.. code-block:: python

    def run(self, argv):
        ret_val = 1
        self.command_options = argv
        try:
            ret_val = super(OpenStackShell, self).run(argv)
            return ret_val
        except Exception as e:
            if not logging.getLogger('').handlers:
                logging.basicConfig()
            if self.dump_stack_trace:
                self.log.error(traceback.format_exc())
            else:
                self.log.error('Exception raised: ' + str(e))

            return ret_val

        finally:
            self.log.info("END return value: %s", ret_val)

여기서도 다시 OpenStackShell.run() 함수로 argv(server list) 인자 값을 넘겨주는데

argv 인자 값은 app(cliff/app.py).App(Class Name)의 함수 run(argv)에게 넘겨주게 된다.

.. code-block:: python

   from cliff import app

   ...

   class OpenStackShell(app.App):

   ...

   def run(self, argv):
       ret_val = 1
       self.command_options = argv
       try:
           ret_val = super(OpenStackShell, self).run(argv)

**cliff/app** 의 run(argv)로 실행

.. code-block:: python

   def run(self, argv):
       """Equivalent to the main program for the application.

       :param argv: input arguments and options
       :paramtype argv: list of str
       """

       try:
           self.options, remainder = self.parser.parse_known_args(argv)

-> self.parser.parse_known_args(argv) 에서

.. code-block:: python

   def parse_known_args(self, args=None, namespace=None):
       if args is None:
           # args default to the system args
           args = _sys.argv[1:]
       else:
           # make sure that args are mutable
           args = list(args)

       # default Namespace built from parser defaults
       if namespace is None:
           namespace = Namespace()

       # add any action defaults that aren't present
       for action in self._actions:
           if action.dest is not SUPPRESS:
               if not hasattr(namespace, action.dest):
                   if action.default is not SUPPRESS:
                       setattr(namespace, action.dest, action.default)

       # add any parser defaults that aren't present
       for dest in self._defaults:
           if not hasattr(namespace, dest):
               setattr(namespace, dest, self._defaults[dest])

       # parse the arguments and exit if there are any errors
       if self.exit_on_error:
           try:
               namespace, args = self._parse_known_args(args, namespace)
           except ArgumentError:
               err = _sys.exc_info()[1]
               self.error(str(err))
       else:
           namespace, args = self._parse_known_args(args, namespace)

       if hasattr(namespace, _UNRECOGNIZED_ARGS_ATTR):
           args.extend(getattr(namespace, _UNRECOGNIZED_ARGS_ATTR))
           delattr(namespace, _UNRECOGNIZED_ARGS_ATTR)
       return namespace, args

환경의 설정 값을 가지고 있는 namespace 정보를 options 변수로 반환해준다.

.. image:: images/picture_3.png

.. image:: images/picture_4.png

그 다음으로 run_subcomman(remainder)로 값을 넘겨준다. # remainder -> server list

.. image:: images/picture_5.png

argv 값을 다시 command_manager.find_command(argv)로 넘겨준다

.. image:: images/picture_6.png

**def find_command(self, argv):**

.. code-block:: python

    def find_command(self, argv):
        """Given an argument list, find a command and
        return the processor and any remaining arguments.
        """
        start = self._get_last_possible_command_index(argv)
        for i in range(start, 0, -1):
            name = ' '.join(argv[:i])
            search_args = argv[i:]
            # The legacy command handling may modify name, so remember
            # the value we actually found in argv so we can return it.
            return_name = name
            # Convert the legacy command name to its new name.
            if name in self._legacy:
                name = self._legacy[name]

            found = None
            if name in self.commands:
                found = name
            else:
                candidates = _get_commands_by_partial_name(
                    argv[:i], self.commands)
                if len(candidates) == 1:
                    found = candidates[0]
            if found:
                cmd_ep = self.commands[found]
                if hasattr(cmd_ep, 'resolve'):
                    cmd_factory = cmd_ep.resolve()
                else:
                    # NOTE(dhellmann): Some fake classes don't take
                    # require as an argument. Yay?
                    arg_spec = inspect.getfullargspec(cmd_ep.load)
                    if 'require' in arg_spec[0]:
                        cmd_factory = cmd_ep.load(require=False)
                    else:
                        cmd_factory = cmd_ep.load()
                return (cmd_factory, return_name, search_args)
        else:
            raise ValueError('Unknown command %r' %
                             (argv,))

found = server list 값을 넣은 후, commands[server list]로 넘겨주어 반환되는 값에 cmd_ep에 server list에 엔드 포인터 값이 담기게 되는데.

server list에 대한 엔드 포인터 값을 받을 수 있었던 이유는 아래의 절차에 의한다

.. image:: images/picture_7.png

find_command 함수는 command_manager 클래스의 있다.

.. code-block:: python

   def run_subcommand(self, argv):
       try:
           subcommand = self.command_manager.find_command(argv)

이때 생성자의 의해 커맨드 명령어에 대한 정보를 가져오게 되는데

다음의 주석을 보면.

.. code-block:: html

   param namespace: String containing the entrypoint namespace for the
   plugins to be loaded. For example, ``'cliff.formatter.list'``.

   플러그인 cliff.formatter.list에서 namespace(cli)의 파라미터(server list)에 대한 처리를 하기 위해 namespace(cli) 엔드포인터 목록을 로딩하는 것 같다.


.. image:: images/picture_8.png

.. image:: images/picture_9.png

이 처리가 끝난 후 commands에는 엔드 포인터 값이 담겨져 있는데

server list의 경우 openstackclient.compute.v2.server.ListServer 에서 처리하는 것을 알 수 있다.

.. image:: images/picture_10.png

.. image:: images/picture_11.png

cmd_factory에는 openstackclient.compute.v2.server.ListServer 엔드 포인트 값이 담겨서 반환된다.

.. image:: images/picture_12.png

.. image:: images/picture_13.png

2. server list  라는 명령어를 처리하는 파일은 무엇인가?
**********************************************************************

따라서 server 파일의 ListServer 함수에서 처리한다는 것을 알 수 있다.

.. image:: images/picture_15.png

3. openstackcli 는 어떻게 nova api 주소를 알아내나요?
**********************************************************************

위로 다시 스크롤을 쭉 올리면 -- **cliff/app 의 run(argv)로** 실행 현재 Namespace 값이 None일 경우
Namespace() 함수로 사용자의 환경 변수를 가져오게 된다.

따라서 우선은 http://211.37.148.129/identity 주소에서 인증을 통해 얻을 수 있을 것이다.

.. image:: images/picture_16.png

다음으로 1,2 과제를 진행하였을 때. server list 엔드 포인트 값을 가져왔었고

find_command 함수가 다음 값을 반환하는 것으로 끝났었는데 여기서 반환된 값은 app.py의 subcommand로 담기게 된다.

.. code-block:: python

   def find_command(self, argv):

   ...

   return (cmd_factory, return_name, search_args)

cmd_parser = cmd.get_parser(full_name) 이 실행되며

.. image:: images/picture_17.png

.. image:: images/picture_18.png

openstackclient.compute.v2.server.ListServer 에서 구문을 분석하여 리턴받는다.

.. image:: images/picture_19.png

openstackclient.compute.v2.server.ListServer

이후로 디버깅으로 계속 분석해보았지만 **API에 대한 값이 담긴 매개 변수를 찾을 수 없었다**

**하지만 log를 살펴보다 compute->openstackclient.compute.client를 살펴볼 수 있었다.**

.. image:: images/picture_20.png

compute.client 의 코드에서 LOG.debug 정보를 출력하는 코드가 있다.

.. image:: images/picture_21.png

따라서 openstack server list --debug 명령어를 입력하여 아래의 정보를 얻을 수 있었다.

.. image:: images/picture_22.png

그렇기에 openstackclient-> compute_v2 -> APIv2 에서 API에 대한 처리를 하며

.. image:: images/picture_23.png

반환되는 Client 값에는

4. openstackcli 는 어떻게 nova api 주소를 알아내나요?
**********************************************************************

**http://211.37.148.129/compute/v2.1** NOVA API 처리하는 주소가 있다

.. image:: images/picture_24.png

5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
**********************************************************************

openstack server list 명령어는 다음의 테이블을 출력해준다.

.. image:: images/picture_25.png

위의 테이블 형식을 찾기 위해 아래의 명령어를 사용하였다.

.. code-block:: html

   grep -R "+-*-+"

.. image:: images/picture_26.png

다시 키워드를 formatters로 하여 검색하였으며 display.py: title=output formatters 와 table.py: Output formatters using prettytable가 유력해 보인다.

.. code-block:: html

   grep -R formatters

.. image:: images/picture_27.png

그 중 table의 emit_list를 변경하여 출력하였다.

.. image:: images/picture_28.png

.. image:: images/picture_29.png

