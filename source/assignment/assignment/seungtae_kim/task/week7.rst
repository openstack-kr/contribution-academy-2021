:orphan:

오픈소스 컨트리뷰션 마지막 스프린트 : 테스트 케이스 수정하기 & 커맨드 개발하기
================================================================================================================

공식적으로 이번 주는 오픈소스 컨트리뷰션 코드 & 번역 커밋을 진행하는 마지막 주다.

프로젝트를 깔끔하게 마무리하면서 이번 주를 종료하고 싶었지만, 프로젝트 maintainer가 코드 개선 점에 대해 피드백을 줬고, 테스트 코드를 많이 수정하게 되었다.


.. image:: https://miro.medium.com/max/700/1*zyoeXbcRc_W6NBvX5bQIRg.png
   :width: 1000px

일전에 해결한 버그에 대해 검증할 수 있는 테스트 코드를 요구했고 그에 따라 test_configuration.py에 테스트 코드를 추가했다.

해당 에러를 재현한 결과, 버그라는 것이 확인되었고 추석 전의 팀 미팅에서 내가 해당 이슈를 맡아서 해결하겠다는 이야기를 하고 나서 2주간 길고 긴 버그 해결 과정에 착수하게 되었다.

.. code-block:: python

    @mock.patch("keystoneauth1.loading.base.get_plugin_options",
                return_value=opts)
    def test_show_mask_with_cloud_config(self, m_get_plugin_opts):
        arglist = ['--mask']
        verifylist = [('mask', True)]
        self.app.client_manager.configuration_type = "cloud_config"
        cmd = configuration.ShowConfiguration(self.app, None)
        parsed_args = self.check_parser(cmd, arglist, verifylist)
        columns, data = cmd.take_action(parsed_args)
        self.assertEqual(self.columns, columns)
        self.assertEqual(self.datalist, data)
    @mock.patch("keystoneauth1.loading.base.get_plugin_options",
                return_value=opts)
    def test_show_mask_with_global_env(self, m_get_plugin_opts):
        arglist = ['--mask']
        verifylist = [('mask', True)]
        self.app.client_manager.configuration_type = "global_env"
        column_list = (
            'identity_api_version',
            'password',
            'region',
            'token',
            'username'
        )
        datalist = (
            fakes.VERSION,
            configuration.REDACTED,
            fakes.REGION_NAME,
            configuration.REDACTED,
            fakes.USERNAME,
        )
        cmd = configuration.ShowConfiguration(self.app, None)
        parsed_args = self.check_parser(cmd, arglist, verifylist)
        columns, data = cmd.take_action(parsed_args)
        self.assertEqual(column_list, columns)
        self.assertEqual(datalist, data)

위의 코드는 테스트를 위해 수정된 내역이고, 아래 코드는 테스트 코드에서 인증을 위해 구현된 fakes.py다.

.. code-block:: python

    def __init__(self):
        self.compute = None
        self.identity = None
        self.image = None
        self.object_store = None
        self.volume = None
        self.network = None
        self.session = None
        self.auth_ref = None
        self.auth_plugin_name = None
        self.network_endpoint_enabled = True
        self.compute_endpoint_enabled = True
        self.volume_endpoint_enabled = True
        self.configuration_type = "cloud_config"
    def get_configuration(self):
        config = {
            'region': REGION_NAME,
            'identity_api_version': VERSION,
        }
        if self.configuration_type == "cloud_config":
            config["auth"] = {
                'username': USERNAME,
                'password': PASSWORD,
                'token': AUTH_TOKEN,
            }
        elif self.configuration_type == "global_env":
            config["username"] = USERNAME
            config["password"] = PASSWORD
            config["token"] = AUTH_TOKEN
        return config

이에 따라 테스트를 진행하기 위해 fake 인증 값을 넣어줘야하는데, fakes.py에 인증 값도 수정해서 넣어줬고, 오픈스택만의 CI 테스트 툴인 Zuul에서 아무 문제 없이 통과하는 것을 확인했다.
(물론 나 혼자만의 업적은 아니고, 멘토님이 옆에서 방향성을 많이 잡아주신 덕분에 수월하게 해결할 수 있었다)

.. image:: https://miro.medium.com/max/700/1*uzrMDlU_HhxmkDH0C3ANSQ.png
   :width: 1000px

그리고 문서에 폴더 경로가 잘못 언급되어 있는 사항에 대해 수정한 것은 merge되어서 결과가 반영되었다.

오픈스택을 하면서 서비스가 실행되는 파일 구조라던가 로직이 조금씩 이해가 되는 느낌이고 버그 개선이라 기능 추가 등에 대한 내용들이 감이 하나씩 잡혀간다.

그리고 컨트리뷰션의 마지막을 장식하기 위해 glance에는 구현되어 있지만 openstack 커맨드에는 구현되어 있지 않던 task-list 명령어 구현 작업을 시작했고, 처음 일주일은 무척 해멨지만, 결과적으로 테스트 코드를 구현하기 전에 커맨드 구현을 완선했다.

.. image:: https://miro.medium.com/max/700/1*3rezvBxhlT_FYNuFA4pDkw.png
   :width: 1000px

.. code-block:: python

    class TaskImage(command.Lister):
        _description = _("Retrieve a listing of Task objects.")
        def get_parser(self, prog_name):
            parser = super(TaskImage, self).get_parser(prog_name)
            parser.add_argument(
                '--sort-key',
                metavar="<key>[:<direction>]",
                default='name:asc',
                help=_("Sort output by selected keys and directions(asc or desc) "
                       "(default: name:asc), multiple keys and directions can be "
                       "specified separated by comma"),
            )
            parser.add_argument(
                "--page-size",
                metavar="<size>",
                help=argparse.SUPPRESS,
            )
            parser.add_argument(
                '--type',
                metavar='<type>',
                choices=[
                    'import', 'export', 'clone'
                ],
                help=_('Filter tasks by type'),
            )
            parser.add_argument(
                '--status',
                metavar='<status>',
                choices=[
                    "pending", "processing", "success", "failure"
                ],
                default=None,
                help=_("Filter tasks based on status.")
            )
            return parser
        def take_action(self, parsed_args):
            image_client = self.app.client_manager.image
            columns = (
                "id",
                "type",
                "status",
                "owner_id"
            )
            column_headers = (
                "ID",
                "Type",
                "Status",
                "Owner"
            )
            data = image_client.tasks()
            return (
                column_headers,
                (utils.get_item_properties(
                    s,
                    columns,
                    formatters=_formatters,
                ) for s in data)
            )
.. image:: https://miro.medium.com/max/700/1*-u5M1ofcAFAGA51ZiN2Y0g.png
   :width: 1000px

Task-api가 다행히 openstack-sdk에 구현되어 있던 덕분에 문제를 잘 해결할 수 있었고, 만약 sdk에 구현되어 있는 것을 확인 못했다면 구현해야하는 작업이 산더미 같이 있었을 것이다.

팀원들과 적극적인 소통 덕분에 일이 수월하게 풀렸고, 잘하면 컨트리뷰션 기간 동안 커맨드를 스스로 개발할 수 있는 능력까지 갖출 수 있게 될 것이다.

후기
------------------------------------------------

스스로도 아직까지 믿기지 않는다.

- 커맨드라는 기능 개발까지 내가 참여할 역량이 된다는 것,
- 그리고 버그 리포트를 날려서 서비스에 문제가 있는 부분을 증명할 수 있다는 점
- 마지막으로 개발자들과 소통하면서 프로젝트에 참여할 수 있다는 점

등을 배울 수 있었고, 올해 openstack의 오픈소스 컨트리뷰션 파트에 참여하길 정~~~~말 잘했다!