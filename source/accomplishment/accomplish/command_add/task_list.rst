:orphan:

================================================================================================================
Command 개발: openstack image task list
================================================================================================================

인프라 구축 오픈소스 프로젝트 오픈스택의 커맨드 라인 만들기

오픈소스 컨트리뷰션의 코드 & 문서 번역 활동 기간이 끝났다.

오픈소스 컨트리뷰션은 내게 의미가 참 큰 활동이다.

개발자로 커리어 방향성을 잡고, 그 동안 많은 활동들을 했지만 엔터프라이즈 급의 큰 프로젝트에 참여할 기회는 좀처럼 없었기 때문에 이번 컨트리뷰션에는 조금 규모가 큰 프로젝트에 참여하기로 목표했고, 다행이도 그 목표에 성공적으로 도달 할 수 있었다.

.. image:: https://miro.medium.com/max/1400/1*RwYVEgK3MhByHZC1zfVb8g.png
   :width: 1000px

무엇보다 문서 설명 오류나 버그 리포트를 하는 과정도 즐거웠지만, 기능을 처음부터 끝까지 빌드하는 것은 어려운 과제 중 하나였고, 솔직히 이번 컨트리뷰션 기간 동안 해낼 수 있을 것이라 믿지 않았다.

그럼에도 좋은 멘토님들과 멘티님들을 만나 자극받고 해낼 수 있었고 생각 이상의 성과를 만들 수 있었다.


1. 구현 코드 설명
--------------------------------------------------------------

내가 이번에 구현한 image task list 라는 커맨드는 오픈스택의 컴포넌트 중 OS image를 관리하는 프로젝트인 glance에서 제공하는 커맨드 중 하나다.

glance에서는 OS image를 관리하는 만큼 클라우드 인프라를 사용하는 유저 중 악성 유저가 의도적으로 os image를 업로드하는 대신 사진 파일 등과 같이 os와는 전혀 상관 없는 이미지를 올려서 서비스를 사용하는데 지장을 줄 수 있다.

때문에 이런 상황을 방지하기 위해서 task 라는 queue를 별도로 구성해서 os image로 올라가기 전에 이를 검증하는 단계를 구성할 수 있는데 이것이 내가 구현한 task-list다.

이 task-list는 expire date가 설정되어 있어 해당 큐에서 task 들은 영구적으로 queue에 존재하는 것이 아니라 시간이 지나면 모두 queue에서 빠져나가 이미지 파일로 등록이 된다.

이 task-queue를 구현하는 것은 내가 한 것이 아니고, 이미 구현된 것에 query를 보내서 상태 값을 가져오는 것이고 사실상 테스트 코드를 어떻게 구현해서 작업하는지가 가장 관건이었다.

.. code-block:: python

    class TaskImage(command.Lister):
    _description = _("Retrieve a listing of Task objects.")

    def get_parser(self, prog_name):
        parser = super(TaskImage, self).get_parser(prog_name)

        parser.add_argument(
            '--sort-key',
            metavar="<key>[:<field>]",
            help=_("Sorts the response by one of the following attributes: "
                   "created_at, expires_at, id, status, type, updated_at. "
                   "Default is created_at. "
                   "multiple keys and directions can be "
                   "specified separated by comma"),
        )

        parser.add_argument(
            '--sort-dir',
            metavar="<key>[:<direction>]",
            help=_("Sort output by selected keys and directions(asc or desc) "
                   "(default: name:desc), multiple keys and directions can be "
                   "specified separated by comma"),
        )

        parser.add_argument(
            "limit",
            metavar="<num-tasks>",
            type=int,
            help=argparse.SUPPRESS,
        )

        parser.add_argument(
            '--type',
            metavar='<type>',
            choices=[
                'import'
            ],
            help=_("Filters the response by a task type. "
                   "A valid value is import. "
                   ),
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

        kwargs = {}
        copy_attrs = ("sort_key", "sort_dir", "limit", "type", "status")
        for attr in copy_attrs:
            if attr in parsed_args:
                val = getattr(parsed_args, attr, None)
                if val is not None:
                    # Only include a value in kwargs for attributes that are
                    # actually present on the command line
                    kwargs[attr] = val

        data = image_client.tasks(**kwargs)

        return (
            column_headers,
            (utils.get_item_properties(
                s,
                columns,
                formatters=_formatters,
            ) for s in data)
        )


위의 코드는 내가 직접 구현한 glance task-list 커맨드를 openstack image task list로 작업한 내용이고, 기존의 task-list에 sort-key, sort-dir, type, status, limit라는 4가지 옵션을 줄 수 있어서 커맨드에서 각 옵션을 파싱해온 후, 이것을 openstack sdk로 커맨드 명령 값을 보내줘서 결과를 받아온다.

아마 이 글을 읽는 사람이 오픈스택을 어느 정도 사용해 본 사람이라면 알겠지만, openstack client에 있는 v2.image.py에 있는 커맨드 코드만 1000줄이 가뿐히 넘기 때문에 코드를 읽어서 내용을 이해하는 것에만 시간이 꽤 들고, image.py 만이 아니라 sdk의 _proxy.py, task.py 등 연관 코드가 꽤 많기 때문에(다 합치면 아마 대략 확인해야하는 코드가 최소 3000줄은 되지 않을까… 이미 구현되어 있는 코드가 있다면, 그 코드를 기반으로 커맨드를 구성해야하고, 구현된 코드가 없다면 짜야하는 코드가 그 많큼 만기 때문이다.) 코드를 실질적으로 작성하는 것보다 각 커맨드들이 어떻게 구현되어 있는지, 작업 패턴 등을 확인하는데 꽤 많은 시간을 들였다.

위에서 각 커맨드들이 정상적으로 작동하는 것을 확인하고 나서, 기능 단위로 unit test 코드를 추가해줘야한다.

아무리 자신의 코드에 확신이 있어도, 프로젝트의 관리자의 입장에서 테스트 코드 없이 추가된 기능을 merge해달라고 하면, 해당 커밋을 reject할 가능성이 크다.

따라서 기능 추가 또는 버그 리포트를 보내야한다면, 거의 모든 상황에 테스트 코드를 포함시켜줘야하기 때문에 나도 아래와 같이 테스트 코드를 구성해서 커밋을 진행했다.

.. code-block:: python

    class TestTaskList(TestTask):

    _data = task_fakes.FakeTask.create_one_task()

    columns = (
        'ID', 'Type', 'Status', 'Owner'
    )

    datalist = (
        _data.id,
        _data.type,
        _data.status,
        _data.owner_id
    )

    def setUp(self):
        super(TestTaskList, self).setUp()

        self.api_mock = mock.Mock()
        self.api_mock.side_effect = [
            [self._data], [],
        ]

        self.client.tasks = self.api_mock

        # Get the command object to test
        self.cmd = image.TaskImage(self.app, None)

    def test_task_list_no_options(self):
        arglist = [
        ]
        verifylist = [
            ('sort_key', None),
            ('sort_dir', None),
            ('limit', None),
            ('type', None),
            ('status', None),
        ]
        parsed_args = self.check_parser(self.cmd, arglist, verifylist)

        # In base command class ShowOne in cliff, abstract method take_action()
        # returns a two-part tuple with a tuple of column names and a tuple of
        # data to be shown.
        columns, data = self.cmd.take_action(parsed_args)

        self.client.tasks.assert_called_with(
        )

        self.assertEqual(self.columns, columns)
        self.assertCountEqual(self.datalist, *data)

    def test_task_list_sort_key_option(self):
        arglist = ['--sort-key', 'created_at']
        verifylist = [('sort_key', 'created_at')]
        parsed_args = self.check_parser(self.cmd, arglist, verifylist)

        # In base command class Lister in cliff, abstract method take_action()
        # returns a tuple containing the column names and an iterable
        # containing the data to be listed.
        columns, data = self.cmd.take_action(parsed_args)

        self.client.tasks.assert_called_with(
            sort_key=parsed_args.sort_key
        )

        self.assertEqual(self.columns, columns)
        self.assertCountEqual(self.datalist, *data)

    def test_task_list_sort_dir_option(self):
        arglist = ['--sort-dir', 'desc']
        verifylist = [('sort_dir', 'desc')]
        parsed_args = self.check_parser(self.cmd, arglist, verifylist)

        # In base command class Lister in cliff, abstract method take_action()
        # returns a tuple containing the column names and an iterable
        # containing the data to be listed.
        columns, data = self.cmd.take_action(parsed_args)

        self.client.tasks.assert_called_with(
            sort_dir=parsed_args.sort_dir
        )

        self.assertEqual(self.columns, columns)
        self.assertCountEqual(self.datalist, *data)

    def test_task_list_page_size_option(self):
        ret_size = 1
        arglist = [
            '--limit', str(ret_size),
        ]
        verifylist = [
            ('limit', ret_size),
        ]
        parsed_args = self.check_parser(self.cmd, arglist, verifylist)

        columns, data = self.cmd.take_action(parsed_args)

        self.client.tasks.assert_called_with(
            page_size=parsed_args.limit,
        )

        self.assertEqual(self.columns, columns)
        self.assertEqual(ret_size, len(tuple(data)))

    def test_task_list_type_option(self):

        arglist = ['--type', self._data.type]
        verifylist = [('type', self._data.type)]
        parsed_args = self.check_parser(self.cmd, arglist, verifylist)

        # In base command class Lister in cliff, abstract method take_action()
        # returns a tuple containing the column names and an iterable
        # containing the data to be listed.
        columns, data = self.cmd.take_action(parsed_args)
        self.client.tasks.assert_called_with(
            type=self._data.type,
        )

        self.assertEqual(self.columns, columns)
        self.assertCountEqual(self.datalist, *data)

    def test_task_list_status_option(self):

        arglist = ['--status', self._data.status]
        verifylist = [('status', self._data.status)]
        parsed_args = self.check_parser(self.cmd, arglist, verifylist)

        # In base command class Lister in cliff, abstract method take_action()
        # returns a tuple containing the column names and an iterable
        # containing the data to be listed.
        columns, data = self.cmd.take_action(parsed_args)
        self.client.tasks.assert_called_with(
            status=self._data.status,
        )

        self.assertEqual(self.columns, columns)
        self.assertCountEqual(self.datalist, *data)

솔직히 테스트 코드 짜기 전까지는 엄청 막막했는데 막상 짜고 나니까 어안이 벙벙했다.

하단의 fakes.py까지 구성하고 테스트 결과가 모두 통과한 것을 확인했을 때가 새벽 4시 10분이었는데, 이날 따라 괜히 오기가 붙어서 분명히 다 해결할 수 있을 것 같다는 마음가짐으로 커밋을 완료할 수 있었다.

.. code-block:: python

    task_id = "53326486-2089-4679-8d24-0b09192ad2d8"
    task_owner = "1565f71789534d35b188781d7ec3e30e"
    task_type = "import"
    task_status = "pending"

    TASK = {
        'id': task_id,
        'type': task_type,
        'owner': task_owner,
        'status': task_status,
    }

    class TestTaskv2(utils.TestCommand):

        def setUp(self):
            super(TestTaskv2, self).setUp()

            self.app.client_manager.image = FakeImagev2Client(
                endpoint=fakes.AUTH_URL,
                token=fakes.AUTH_TOKEN,
            )

            self.app.client_manager.identity = identity_fakes.FakeIdentityv3Client(
                endpoint=fakes.AUTH_URL,
                token=fakes.AUTH_TOKEN,
            )


    class FakeTask(object):
        """Fake one or more tasks.
        TODO(lsmman): Currently, only image API v2 is supported by this class.
        """

        @staticmethod
        def create_one_task(attrs=None):
            """Create a fake image.
            :param Dictionary attrs:
                A dictionary with all attrbutes of image
            :return:
                A FakeResource object with id, name, owner, protected,
                visibility, tags and size attrs
            """
            attrs = attrs or {}

            # Set default attribute
            task_info = {
                'id': task_id,
                'type': task_type,
                'owner': task_owner,
                'status': task_status,
            }

            # Overwrite default attributes if there are some attributes set
            task_info.update(attrs)

            return task.Task(**task_info)

        @staticmethod
        def create_tasks(attrs=None, count=2):
            """Create multiple fake images.
            :param Dictionary attrs:
                A dictionary with all attributes of image
            :param Integer count:
                The number of images to be faked
            :return:
                A list of FakeResource objects
            """
            tasks = []
            for n in range(0, count):
                tasks.append(FakeTask.create_one_task(attrs))

            return tasks

테스트 코드는 /v2/task에서 실제 인증 값을 가져오는 것은 아니고, 미리 만들어 놓은 fake 인증값을 기반으로 mockup 데이터를 생성한 것을 토대로 테스트 코드를 실행한다.

그러나 이 때 호출하는 정보는 내가 만들어놓은 실제 기능을 기반으로 실행되기 때문에, 아마 내 코드에서 에러가 나는 부분은 없었을 것이라 생각한다.

.. image:: https://miro.medium.com/max/700/1*REadrzguLKz84u_E-mQcMw.png
   :width: 1000px

다 짜고 보니 코드가 350줄 정도 되었고, 이전의 버그 리포트까지 포함하면 오픈스택 코드 중에 400줄 정도를 내가 직접 문제를 해결한 것이 되었다.

오픈 스택 코드를 보면서 문제 해결력이 무척이나 많이 늘었다.

이제 좀 현업에서 일할 수 있을 것 같다는 자신감도 높아졌고, 조금 더 오픈 스택 부분을 깊게 파고 들면 클라우드 인프라에 대한 이해도 많이 높아질 것 같다.

클라우드는 앞으로 계속 발전할 것이니, 이 분야에 대한 이해도 어느 정도는 항상 트렌드를 따라가야 하지 않을까 싶다.

커맨드 구현 마무리 못할까봐 조마조마 했는데, 기어이 이것까지 마무리했다.

(아침에 일어나서 보니 Zuul CI에서 모든 테스트 결과도 통과했다)

아~~~주 흡족했던 오픈소스 컨트리뷰션 활동이었다!


2. Reference
------------------------------------------------

- `내 Gerrit PR 확인하기 <https://review.opendev.org/c/openstack/python-openstackclient/+/813554>`_
