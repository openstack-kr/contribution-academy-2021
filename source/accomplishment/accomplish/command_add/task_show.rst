:orphan:

================================================================================================================
Command 개발: openstack image task show
================================================================================================================

미리 보는 결론
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
명령어를 구현하는 것으로 Openstack의 기능 하나하나가 어떤 방식으로 구현되어 있는지 체득할 수 있었습니다.

결과적으로는 관련 코드도 수천 줄을 읽었다. 어느 때보다 많이 코드를 읽었고, 기여에 필요한 프로세스도 모두 문서를 보고 실습하며 공부했습니다.

기여 활동이라고 여겼던 오픈소스 활동이 개발자 본인에게도 이렇게 크게 도움이 될 지는 몰랐습니다.

좋은 멘토님들과 멘티님들, 메인테이너들과 소통을 통해, 정말 배울 수 있었습니다.

앞으로도 개발을 하며 살아가면서 오픈 소스를 지속하면서 살아가고 싶습니다.


배경
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Openstack은 Python-openstackclient와 OpenstackSDK를 중심으로 각각의 컴포넌트의 Client로 구성된 명령어들을 Opnestack으로 통합하고 있습니다.

그 중 우리는 명령어 아직 통합된 OpenstackSDK에 추가되지 않았던 명령어를 구현하는 데에 주안점을 두고 진행했습니다.

그 중 **Glance** 라는 OS 이미지를 관리하는 컴포넌트의 `task-create, task-list, task-show` 3개의 명령어를 3명의 멘티가 각각 맡아 구현하였습니다.

멘토님과 3명의 멘티가 따로 미팅을 수차례 함께 하면서, 끝내 구현에 성공했습니다.

    .. image:: ../images/command_task_show.png
        :width: 700


기능 설명
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**image task show** 라는 명령어는 그 중 **Glance** 라는 OS 이미지를 관리하는 컴포넌트의 커맨드 중 하나입니다.

glance에서 잘못된 형식의 OS image가 업로드되거나 만들어졌을 때 잘못된 이미지로 플랫폼에서 문제가 될 수 있습니다.
task는 glance에서 새로운 image를 등록하기 전에 OS-Image의 상태와 등록 결과를 저장합니다.

그 중 내가 구현한 명령어는 만들어진 task를 지정해서 세부 사항을 확인할 수 있는 "task show"입니다.


기능 동작
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**명렁어 사용법**

    .. code-block::

        Usage: image task show <Task ID>

**기능 동작 예시**

    .. code-block::

        $ openstack image task show 53326486-2089-4679-8d24-0b09192ad2d8
        +------------+--------------------------------------------+
        | Field      | Value                                      |
        +------------+--------------------------------------------+
        | created_at | 2021-10-14T05:56:20Z                       |
        | expires_at | 2021-10-16T05:56:20Z                       |
        | id         | 53326486-2089-4679-8d24-0b09192ad2d8       |
        | input      | {}                                         |
        | message    | Input does not contain 'import_from' field |
        | owner      | 1565f71789534d35b188781d7ec3e30e           |
        | properties | schema='/v2/schemas/task'                  |
        | status     | failure                                    |
        | type       | import                                     |
        | updated_at | 2021-10-14T05:56:20Z                       |
        +------------+--------------------------------------------+


기능 구현 방법
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Image task show 작성**

    .. code-block::python

        def take_action(self, parsed_args):
            image_client = self.app.client_manager.image

            task = image_client.get_task(parsed_args.task)
            info = _format_task(task)

            return zip(*sorted(info.items()))

\

image_client의 get_task를 호출하고 그 결과로 Task 객체를 반환합니다.
Task 객체를 view로 변환하기 위해 format 함수로 처리하고 sort한 결과를 return 합니다.

크게 3개의 구현체로 이루어져 있습니다.

- image_client의 get_task 함수
- Task 객체
- _format_task 함수

image_client는 client_manager가 keystone에서 openstacksdk의 구현체를 인스턴스화해서 가져오는 객체입니다.

이 부분은 꼭 해당 명령어를 디버깅해가면서 확인해 보셔야 찾을 수 있습니다. `과제 2 참고 <https://openstack-kr-contribution-academy-2021.readthedocs.io/ko/latest/assignment/assignment/seunghyun_lim/task/week2-2_server-list.html>`_

- openstacksdk 프로젝트의 *openstack/image/v2/_proxy.py* 의 get_task의 부분

    .. code-block:: python

        def get_task(self, task):
            """Get task details

            :param task: The value can be the ID of a task or a
                         :class:`~openstack.image.v2.task.Task` instance.

            :returns: One :class:`~openstack.image.v2.task.Task`
            :raises: :class:`~openstack.exceptions.ResourceNotFound`
                     when no resource can be found.
            """
            return self._get(_task.Task, task)

\

- Task 객체

    - _format_task 역시 Task 객체의 member 변수를 dictionary 형태로 반환합니다.
    - Task 객체로 보여줄 수 있는 멤버 변수는 created_at, expires_at, input, message, owner_id, result, schema, status, type 가 있습니다.

    .. code-block:: python

        class Task(resource.Resource):
            resources_key = 'tasks'
            base_path = '/tasks'

            # capabilities
            allow_create = True
            allow_fetch = True
            allow_list = True

            _query_mapping = resource.QueryParameters(
                'type', 'status', 'sort_dir', 'sort_key'
            )

            created_at = resource.Body('created_at')
            expires_at = resource.Body('expires_at')
            input = resource.Body('input')
            message = resource.Body('message')
            owner_id = resource.Body('owner')
            result = resource.Body('result')
            schema = resource.Body('schema')
            status = resource.Body('status')
            type = resource.Body('type')
            updated_at = resource.Body('updated_at')

\

Image 객체와 image_client의 find_image 함수는 각각 openstacksdk로부터 호출된 결과입니다.
(디버깅으로 객체를 인스턴스화한 상태로 보아야 openstackSDK로 연결됨을 확인할 수 있습니다.)


**Task의 유닛 테스트를 위한 Test와 Fake 구현**

- Python-openstackclient에서는 가져온 API의 기능이 해석되어 **Openstack shell commmand로 잘 호출 되는 지** 가 테스트의 핵심입니다.
- 이를 위해 유닛 테스트이 API 호출 부분은 Fake Mock 객체를 사용하여 처리하고 있고 기존에 없던 Fake_task를 구현하였습니다.
- 등록된 Image의 상태와 종류, 이미지를 등록할 owner 프로젝트가 Task를 나타냅니다. 따라서 Task에서 핵심적으로 일치 여부를 테스트 해야하는 부분은 `id, type, owner, status` 로 지정했습니다.


- Fake_task 코드

    .. code-block:: python

        class FakeTaskv2Client(object):

            def __init__(self, **kwargs):

                self.get_task = mock.Mock()
                self.get_task.resource_class = fakes.FakeResource(None, {})
                self.auth_token = kwargs['token']
                self.management_url = kwargs['endpoint']
                self.version = 2.0


        class TestTaskv2(utils.TestCommand):

            def setUp(self):
                super(TestTaskv2, self).setUp()

                self.app.client_manager.image = FakeTaskv2Client(
                    endpoint=fakes.AUTH_URL,
                    token=fakes.AUTH_TOKEN,
                )

                self.app.client_manager.identity = identity_fakes.FakeIdentityv3Client(
                    endpoint=fakes.AUTH_URL,
                    token=fakes.AUTH_TOKEN,
                )


        class FakeTask(object):
            """Fake one or more tasks.
            """

            @staticmethod
            def create_one_task(attrs=None):
                """Create a fake task.

                    :param Dictionary attrs:
                        A dictionary with all attributes of task
                    :return:
                        A FakeResource object with id, type and owner attrs
                """
                attrs = attrs or {}

                # Set default attribute
                task_info = {
                    'id': str(uuid.uuid4()),
                    'type': 'image-name' + uuid.uuid4().hex,
                    'owner': 'image-owner' + uuid.uuid4().hex,
                    'status': 'image-status' + uuid.uuid4().hex,
                }

                # Overwrite default attributes if there are some attributes set
                task_info.update(attrs)
                return task.Task(**task_info)

\

- python-openstackclient의 unit test를 위한 test_image.py 코드

    .. code-block:: python

        class TestTask(task_fakes.TestTaskv2):

            def setUp(self):
                super(TestTask, self).setUp()

                # SDK proxy mock
                self.app.client_manager.image = mock.Mock()
                self.client = self.app.client_manager.image

                self.client.show_task = mock.Mock()


        class TestTaskShow(TestTask):

            _data = task_fakes.FakeTask.create_one_task()

            columns = (
                'id', 'owner', 'status', 'type'
            )

            data = (
                _data.id,
                _data.owner_id,
                _data.status,
                _data.type
            )

            def setUp(self):
                super(TestTaskShow, self).setUp()

                self.client.get_task = mock.Mock(return_value=self._data)

                # Get the command object to test
                self.cmd = image.ShowTask(self.app, None)

            def test_task_show(self):
                arglist = [
                    task_fakes.task_id
                ]
                verifylist = [
                    ('task', task_fakes.task_id),
                ]
                parsed_args = self.check_parser(self.cmd, arglist, verifylist)

                # In base command class ShowOne in cliff, abstract method take_action()
                # returns a two-part tuple with a tuple of column names and a tuple of
                # data to be shown.
                columns, data = self.cmd.take_action(parsed_args)
                self.client.get_task.assert_called_with(
                    task_fakes.task_id
                )

                self.assertEqual(self.columns, columns)
                self.assertCountEqual(self.data, data)

\

- 가장 어려웠던 것은 하나도 내가 계획하지 않았던 거대한 코드를 수정하기 시작하는 것이었습니다.
- 저 또한 유사한 형태의 코드를 프로젝트 내에서 참고하면서 시작하였습니다.
- 그 코드의 목적성에 맞게 고쳐나가고 모르면 동료들과 커뮤니티에 물어가며 진행하였습니다.
- 이 글이 새로이 오픈스택에서 기능 구현하는 분들께 자그마한 도움이 되신다면 좋겠습니다.
- 혹시 질문 있으시면 lsmman07@gmai.com로 메일 주세요.

참고
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- `task show PR 확인하기 <https://review.opendev.org/c/openstack/python-openstackclient/+/813436>`_