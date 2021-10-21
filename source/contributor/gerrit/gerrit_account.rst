:orphan:

OpenStack Gerrit 계정 생성 및 커밋 환경 설정하기
=======================================================================
OpenStack 커뮤니티에 기여하기 위해 Git을 기반으로 코드 리뷰 서비스를 제공하는 Gerrit과 OpenStack에서 발생하는 이슈를 관리하는 Storyboard를 사용하는 방법에 대해 안내해드리겠습니다.


1. Gerrit 계정 생성하기
------------------------------------------------

- Gerrit과 Storyboard를 사용하려면 모두 회원가입이 필요하고, 이 둘의 계정은 Ubuntu One이라는 계정을 생성하면 모두 통합해서 관리하는 것이 가능해집니다.

- 그래서 한 개의 계정을 생성하면 나머지 서비스들은 소셜 로그인 형태로 접근해 사용하는 것이 가능합니다.

- Ubuntu One에 회원가입하고, Gerrit 페이지에서 새로운 컨트리뷰터로서 활동하겠다는 서약에 동의하면, 이제 Gerrit에서 코드 리뷰를 할 수 있는 권한이 생성됩니다.

.. image:: https://miro.medium.com/max/614/1*1ecCnMLqbyxg3iNK0J_Z1A.png
   :width: 1000px

- 위의 새 컨트리터로 등록하는 서명을 클릭하면 아래의 이미지를 볼 수 있습니다.

.. image:: https://miro.medium.com/max/700/1*mNRMRw0oR2RyWzBu0bsqJQ.png
   :width: 1000px

- 서명은 개인으로서 오픈스택 커밋터가 되겠다는 것을 클릭하면 되며, 클릭하고 동의하는 문구까지 작성하면 이제 Gerrit을 이용하기 위한 1차적인 설정은 완료됩니다.

.. image:: https://miro.medium.com/max/441/1*Wr8X2dBmmZ87xcAehWI90w.png
   :width: 1000px

- 참고로 위의 서명을 하는 페이지가 프로필 페이지인데, 본인의 Username도 꼭 해주세요.

- 추후에 터미널로 Gerrit 설정할 때 본인의 Gerrit Username을 반드시 등록해야하기 때문에 해당 설정을 안하고 진행하면, 에러가 발생할 수 있습니다.

2. SSH 키 생성하기
------------------------------------------------

- 먼저 글을 읽는 독자님의 로컬 컴퓨터에 본인이 소유하고 있는 SSH 키 현황을 확인해주세요.

- 명령어는 아래의 코드를 터미널에 실행하면 확인할 수 있습니다.

.. code-block:: console

    > ls -la ~/.ssh

.. image:: https://miro.medium.com/max/626/1*0t12O0l-lSKMUfB7Uu-LPA.png
   :width: 1000px

- 위 명령어를 실행하면 키 파일이 몇개 검색되는데 여기 있는 것 중 하나를 사용하는 것은 아닙니다.

- 여기서 키를 확인하고 나면, 아래 명령어로 독자님의 이메일을 인지하는 키를 생성해 줍니다.

.. code-block:: console

    > ssh-keygen -t rsa -b 4096 -C "your_email@example.com"


- 그럼 아마 파일 이름 및 보안 비밀번호 설정 등을 언급하는데, 외부로 유출할 게 아니라면 전부 엔터를 눌러줍니다. (총 3번)

.. code-block:: console

    Enter a file in which to save the key (/Users/you/.ssh/id_rsa): [Press enter]
    Enter passphrase (empty for no passphrase): [Type a passphrase]
    Enter same passphrase again: [Type passphrase again]

- 그 후에 ~/.ssh/config 경로에 config라는 파일을 생성해서 아래와 같은 내용을 설정해줍니다.

.. code-block:: console

    Host review.opendev.org review
      Hostname review.opendev.org
      Port 29418
      User <your_gerrit_username>
      IdentityFile ~/.ssh/<당신의 SSH Key File Name>

- 아까 Gerrit Username을 세팅하라는 게 여기서도 쓰이기도 하고 뒷 부분에서 계속 쓰일 거라 유저이름은 미리 만들어두는 것이 좋습니다.

- 그 후에 당신이 생성한 SSH 키의 공개키를 아래 명령어로 복사해줍니다.

.. code-block:: console

    > cat ~/.ssh/<당신의 SSH key file.pub>

.. image:: https://miro.medium.com/max/700/1*tmiBFzAJJMiIODg31MRjYg.png
   :width: 1000px

- 위의 키를 복사해서 아까 전 Gerrit 프로필 페이지의 SSH Key라는 곳에 붙여넣고 추가하기 버튼을 눌러줍니다.

.. image:: https://miro.medium.com/max/641/1*9ILZ8ax2DecUHPGRETH4vw.png
   :width: 1000px

- 중요한 건 ssh-rsa ~ <당신의 이메일>까지 모두 복사해서 넣어야 에러가 발생 안하니 주의하시길 바랍니다.

- 여기까지하면 공개키 생성과 Gerrit 계정 생성이 마무리됩니다.

3. Gerrit을 로컬에서 사용하기 위한 git-review 설치하기
------------------------------------------------------------------------------------------------

- 먼저 Mac에서 당신이 Gerrit을 사용하는 상황이라면 아래 명령어를 사용해서 git-review라는 모듈을 설치해줍니다.

.. code-block:: console

    > pip install git-review

- 참고로 MacOS Sierra를 사용하면 에러가 발생할 수 있으니 아래 설정을 참고해주세요.

.. image:: https://miro.medium.com/max/700/1*TJArJknKvbNupk1AAbw8aw.png
   :width: 1000px

- 모듈 설치가 끝났다면, 바로 아래 설정을 추가적으로 해줍니다.

.. code-block:: console

    > git config — global — add gitreview.username “<당신의 Gerrit Username>”

- git-review를 설치하면 당신의 오픈스택 클론 프로젝트가 설치되어 있는 경로로 넘어가서 이 명령어를 수행해주세요.

.. code-block:: console

    > git review -s

- git reivew를 초기화하는 작업인데, 알아서 공개키를 찾아서 인식하고 그 키를 사용할 것인지 물어볼텐데, yes를 해줍니다.

.. image:: https://miro.medium.com/max/700/1*6BE-6XHnbC12b1MBJ8rrwA.png
   :width: 1000px

- 여기까지 했다면, 이제 당신의 Gerrit에 대한 모든 설정은 끝났습니다.


4.Reference
------------------------------------------------

- `Gerrit 컨트리뷰터 서약하기 <https://review.opendev.org/settings/#Agreements>`_

- `Gerrit 설정하기 국문 공식 문서 <https://docs.openstack.org/contributors/ko_KR/common/setup-gerrit.html>`_

- `Git Set up <https://docs.openstack.org/contributors/common/git.html#id1>`_

- `Using Gerrit <https://docs.openstack.org/contributors/code-and-documentation/using-gerrit.html>`_