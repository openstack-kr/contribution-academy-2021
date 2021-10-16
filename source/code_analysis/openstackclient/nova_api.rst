openstackclient에서 nova api version 선택하는법
=====================================================================

1. 테스트 환경
------------------------------
    2021년 10월 15일 작성 기준

    ``openstackclient`` , ``osc-lib`` , ``openstacksdk`` , ``novaclient`` 3개 소스코드 전부 master 브런치

    ``python3.8`` 이용

    ``parameters`` 는 아무 명령어를 입력해도 되는데 중간에 --os-compute-api-version 2.53
    이라는 환경변수를 추가한다.

    ex) (처음) hypervisor list
        (중간) hypervisor list --os-compute-api-version 2.53

--------

2. 코드 분석 시작!
''''''''''''''''''''''''''
처음 시작은 일단 언제 compute(nova) api version이 선택되는지  찾아보자.

    * ``디버그`` 모드로 하고 ``Debugger`` 에서 self.api_version에 compute에 api버전이 찍히는지 찾아보니
      아래 코드에서 나온다!

    ``cliff/app.py`` 의 258L을 실행하면 아래처럼 api version이 뜬다!'

    .. code-block:: python

         39 class App(object):
        ..
        235     def run(self, argv):
        ..
        258         self.initialize_app(remainder)
        ..

    .. code-block:: python

        >>> self.api_version
        {'compute': '2.1', 'identity': '3', 'image': '2', 'network': '2', 'object_store': '1', 'volume': '3'}


    * 계속 안으로 들어가며 self.api_version에 값이 채워지는 부분을 찾아보면

    ``osc-lib/osc_lib/shell.py`` 의 442L에서 값이 채워진다!

    .. code-block:: python

        388     def initialize_app(self, argv):
        ...
        442         self._load_plugins()
        ...

    * 안으로 들어가면 openstackclient/shell.py 의 65L ``_load_plugins`` 라는 함수로 넘어오는데
      바로 여기서 compute의 api version이 선택이 된다!! 이제 분석해보자


    .. code-block:: python

        def _load_plugins(self):
            ...
            for mod in clientmanager.PLUGIN_MODULES:
            # 여기서 clientmanager.PLUGIN_MODULES을 출력해보면
            # 'openstackclient.compute.client'가 첫번째로 나온다.
            # 그럼 처음에 compute api version을 먼저 선택한다는 이야기다!

            # 그리고 이제부터 openstackclient/compute/client.py를 켜놓고 보자!

                default_version = getattr(mod, 'DEFAULT_API_VERSION', None)
                # 위에서 mod 즉, openstackclient/compute/client.py에서
                # DEFAULT_API_VERSION('2.1') 값을 가져온다.

                # 아래는 client.py코드의 일부이다.
                # openstackclient/compute/client.py-------------#
                26 DEFAULT_API_VERSION = '2.1'
                27 API_VERSION_OPTION = 'os_compute_api_version'
                28 API_NAME = 'compute'
                29 API_VERSIONS = {
                30     "2": "novaclient.client",
                31     "2.1": "novaclient.client",
                32 }
                #-----------------------------------------------#

                option = mod.API_VERSION_OPTION.replace('os_', '', 1)
                # client.py의 API_VERSION_OPTION 값에서 'os_'를 1번만 지워주고 option에 넣는다.

                version_opt = str(self.cloud.config.get(option, default_version))
                # 여기서 parameters에 --os-compute-api-version 2.53을 추가해주면
                # 2.53이 version_opt에 저장이 되고 아니면 default_version('2.1')이 저장이 된다.


                if version_opt:
                    api = mod.API_NAME
                    # client.py의 API_NAME('compute')를 저장하고

                    self.api_version[api] = version_opt
                    # self.api_version에 compute: 2.1을 저장한다.
                    # 바로 여기서 저장이 되는거다!!!
                    # 하지만 저장만 하고 끝은 아니고 이제 버전이 과연 유효한 버전인지 아래에서 검사를 한다.

                    skip_old_check = False
                    mod_check_api_version = getattr(mod, 'check_api_version', None)

                    if mod_check_api_version:
                        skip_old_check = mod_check_api_version(version_opt)
                        # version_opt을 넘겨주고 결과값을 가져온다.

                        # openstackclient/compute/client.py------------------------------------------------#
                        # 117 def check_api_version(check_version):
                                ...
                                _compute_api_version = api_versions.get_api_version(check_version)

                                if not _compute_api_version.is_latest():
                                # 버전이 양의 무한대로 가는지 검사해서 정상이면 if문 실행

                                    if _compute_api_version > api_versions.APIVersion("2.0"):
                                    # 여기서 알 수 있는건 compute(nova) api version은 최소 2.0보단 높아야한다!

                                        if not _compute_api_version.matches(
                                            novaclient.API_MIN_VERSION,
                                            novaclient.API_MAX_VERSION,
                                        ):
                                        # novaclient.API_MIN과 MAX_VERSION은 아래 코드에서 가져온다.
                                        # python-novaclient/novaclient/__init__.py------------------------------------#
                                            ...
                                            API_MIN_VERSION = api_versions.APIVersion("2.1")
                                            ...
                                            API_MAX_VERSION = api_versions.APIVersion("2.90")
                                        #-----------------------------------------------------------------------------#

                                        # _compute_api_version.matches에 들어가보면
                                        # python-novaclient/novaclietn/api_versions.py의 matches함수가 나온다.
                                        # api_versions.py-------------------------------------------------------------#
                                            def matches(self, min_version, max_version):

                                                if self.is_null():
                                                    raise ValueError(_("Null APIVersion doesn't support 'matches'."))
                                                if max_version.is_null() and min_version.is_null():
                                                    return True
                                                elif max_version.is_null():
                                                    return min_version <= self
                                                elif min_version.is_null():
                                                    return self <= max_version
                                                else:
                                                    return min_version <= self <= max_version
                                                    # 설정한 compute(nova) api version이 2.1과 2.90사이인지 체크한다.
                                                    # 여기서 또 알 수 있는것은 현재 지원하는 api 버전은 2.1부터 2.90이다!
                                        #-----------------------------------------------------------------------------#


                                            msg = _("versions supported by client: %(min)s - %(max)s") % {
                                                "min": novaclient.API_MIN_VERSION.get_string(),
                                                "max": novaclient.API_MAX_VERSION.get_string(),
                                            }
                                            raise exceptions.CommandError(msg)
                                            # 여기는 설정한 버전이 2.1~2.90사이에 버전이 아니면 에러메세지를 출력하는 부분이다.

                                return True
                                # 설정한 api version이 유효하면 True를 반환한다.
                        #---------------------------------------------------------------------------------------#

                    mod_versions = getattr(mod, 'API_VERSIONS', None)
                    # 위 코드는 정확히 뭘 위해서 사용하는지 아직은 잘 모르겠다.

                    if not skip_old_check and mod_versions:
                    # 만약 api version이 유효하지 않으면 실행되어 에러메세지를 출력한다.

                        if version_opt not in mod_versions:
                            sorted_versions = sorted(
                                mod.API_VERSIONS.keys(),
                                key=lambda s: list(map(int, s.split('.'))))
                            self.log.warning(
                                "%s version %s is not in supported versions: %s"
                                % (api, version_opt, ', '.join(sorted_versions)))

                    version = '.v' + version_opt.replace('.', '_').split('_')[0]
                    cmd_group = 'openstack.' + api.replace('-', '_') + version
                    self.command_manager.add_command_group(cmd_group)
                    self.log.debug(
                        '%(name)s API version %(version)s, cmd group %(group)s',
                        {'name': api, 'version': version_opt, 'group': cmd_group}
                    )


    * 이렇게 되어있어서 우리가 기본적으론 compute(nova) api version은 ``2.1`` 로 설정이 된다.
