OpenStack 팀 2주차 진행사항 : openstack server list 명령어 동작 원리 파악
==========================================================================

2주차 2번째 과제에 대한 풀이 진행 사항입니다.

1. 인자로 입력받은 server list 를 어떻게 구별해내는가?
2. server list  라는 명령어를 처리하는 파일은 무엇인가?
3. openstackcli 는 어떻게 nova api 주소를 알아내나요?
4. nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )
5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?

1. 인자로 입력받은 server list 를 어떻게 구별해내는가?
--------------------------------------------------------

- 우선 순서대로 `openstackclient/shell.py` 에서 명렁어로 `server list` 라는 문구를 받게 되 파이썬 `sys` 모듈에 의해 `sys.argv[1:]`  입력된 것을 처리하게 됩니다.

- 그리고 해당 입력 값을 `OpenStackShell().run(argv)` 에서 처리해주게 되는데, 이 클래스와 `run` 함수는 python-openstackclient 레포의 의존성 모듈인 `osc_lib/shell.py` 에서 실행되게 됩니다.

    .. code-block:: python

        # osc_lib/shell.py/OpenStackShell class
        def run(self, argv):
            ret_val = 1
            self.command_options = argv
            try:
                ret_val = super(OpenStackShell, self).run(argv) # run은 cliff 모듈의 app 클래스의 메소드임.
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

- 다만, 여기서 주목할 것이 있습니다. 5번째 라인의 `ret_val` 이라는 변수에 상독된 값에 `run` 메소드가 한 번 더 붙었습니다.

- 이는 cliff (Command Line Interface Formulation Framework)의 약자로 커맨드 명령어를 빌드해주는 파이썬 프레임워크입니다.

- 해당 프레임워크의 app.py로 넘어가 확인하게 되면, 코드의 역할을 명확하게 다 이해하는 것은 아니지만 run 함수가 마찬가지로 있는 것을 볼 수 있습니다.

    .. code-block:: python

        # cliff/app.py
        def run(self, argv):
            """Equivalent to the main program for the application.

            :param argv: input arguments and options
            :paramtype argv: list of str
            """

            try:
                self.options, remainder = self.parser.parse_known_args(argv)
                self.configure_logging()
                self.interactive_mode = not remainder
                if self.deferred_help and self.options.deferred_help and remainder:
                    # When help is requested and `remainder` has any values disable
                    # `deferred_help` and instead allow the help subcommand to
                    # handle the request during run_subcommand(). This turns
                    # "app foo bar --help" into "app help foo bar". However, when
                    # `remainder` is empty use print_help_if_requested() to allow
                    # for an early exit.
                    # Disabling `deferred_help` here also ensures that
                    # print_help_if_requested will not fire if called by a subclass
                    # during its initialize_app().
                    self.options.deferred_help = False
                    remainder.insert(0, "help")
                self.initialize_app(remainder) # 제일 중요한 부분!
                self.print_help_if_requested()
            except Exception as err:
                if hasattr(self, 'options'):
                    debug = self.options.debug
                else:
                    debug = True
                if debug:
                    self.LOG.exception(err)
                    raise
                else:
                    self.LOG.error(err)
                return 1
            except KeyboardInterrupt:
                return _SIGINT_EXIT
            result = 1
            if self.interactive_mode:
                result = self.interact()
            else:
                try:
                    result = self.run_subcommand(remainder)
                except KeyboardInterrupt:
                    return _SIGINT_EXIT
            return result

- 다른 명령어 부분들은 크게 이해하고 넘어갈 필요가 없지만, 앱을 초기화하는 함수가 있습니다.

- self.initialize_app(remainder) 타고 가면, 아래의 2개의 함수가 마찬가지로 호출되고 있는 것을 알 수 있습니다.

    .. code-block:: python

        self._load_plugins()

        self._load_commands()

- 이 함수들은 다시 python-openstackclient 폴더로 넘어가서 shell.py에서 실행되는데

    .. code-block:: python

       version = '.v' + version_opt.replace('.', '_').split('_')[0]
       cmd_group = 'openstack.' + api.replace('-', '_') + version # 커맨드 라인이 인식되는 부분

- 이 두줄을 통해 커맨드를 인식하는 것을 확인할 수 있습니다.

2. server list  라는 명령어를 처리하는 파일은 무엇인가?
--------------------------------------------------------------

- `openstackclient/compute/v2/server.py` 의 `class ListServer(command.Lister)` 클래스에서 최종적으로 명령어를 처리하게 됩니다.

.. code-block:: python

        ...
        columns = (
            'ID',
            'Name',
            'Status',
            'Networks',
            'Image Name',
            'Flavor Name',
            'created', # 승태 수정 영역
            "tenant_id",
        )
        column_headers = (
            'ID',
            'Name',
            'Status',
            'Networks',
            'Image',
            'Flavor',
            'Created_At', # 승태 수정 영역
            "Project_ID",
        )
        ...

- ListServer에서 take_action 함수에서 해당 일을 처리하며, 3주차 과제와 연결되는 부분으로 해결됩니다.

3. openstackcli 는 어떻게 nova api 주소를 알아내나요?
---------------------------------------------------------

- API 주소를 찾는 과정을 보기 위해 가장 먼저 아래의 명령어 2개를 실행해봤습니다.

.. code-block::

    > openstack catalog list
    > openstack endpoint list

.. image:: https://miro.medium.com/max/700/1*VAzKmr904HwFhuJo9eWQ4w.png
   :width: 1000px

- 이에 따라 이렇게 표 형태로 결과 값을 확인했고, python-openstackclient/identity/v2_0/endpoint.py 경로에서 take_action 함수가 API 주소를 인지해서 가져오는 것으로 이해했습니다.

4. nova 의 어떤 API를 호출하여 결과를 받아오나요? ( 어떤 URI 를 호출하나요? )
-------------------------------------------------------------------------------

- 3번 문제를 해결하면서 가져오는 Nova API 주소는 아래와 같은 형태를 결과로 받아온다고 결론지었습니다.

.. code-block::

    /compute/v2.1/<instance address>

5. 결과를 이쁘게 table 형식으로 출력해주는 함수는 무엇일까요?
--------------------------------------------------------------

- cliff 모듈에서 formatter라는 폴더가 있고 (차트 형태를 구성해주는 파일) 거기서 table.py가 이 일을 수행해주는 것으로 이해했습니다.

- 여기서 emit_list라는 함수에 print를 넣어주면 결과에 함께 내 프린트 결과물이 출력되는 것을 볼 수 있습니다.

.. image:: https://miro.medium.com/max/563/1*ABbu7PXVkdg3CparfVK9NQ.png
   :width: 1000px

- 결과 확인

.. image:: https://miro.medium.com/max/700/1*ZCxhYS8N38r-fde87Hcvjg.png
   :width: 1000px
