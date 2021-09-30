:orphan:

OpenStack 팀 4주차 진행사항 : Gerrit & Storyboard 설정하기
=======================================================================


1. Gerrit 계정 설정 및 SSH 키 생성
------------------------------------------------

1-1) 계정 생성 후, 컨트리뷰터 서명하기
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- 사실 Gerrit과 Storyboard를 사용하려면 모두 회원가입이 필요하고, 이 둘의 계정은 Ubuntu One이라는 계정을 생성하면 모두 통합해서 관리하는 것이 가능해집니다.

- 그래서 한 개의 개정을 생성하고 소셜 로그인 형태로 접근해 사용해야합니다..

- 그리고 새로운 컨트리뷰터로서 활동하겠다는 서약에 동의하면, 이제 Gerrit에서 코드 리뷰를 할 수 있는 권한이 생성됩니다..

.. image:: https://miro.medium.com/max/614/1*1ecCnMLqbyxg3iNK0J_Z1A.png
   :width: 1000px

- 위의 새 컨트리터로 등록하는 서명을 클릭하면 아래의 내용을 볼 수 있습니다.

.. image:: https://miro.medium.com/max/700/1*mNRMRw0oR2RyWzBu0bsqJQ.png
   :width: 1000px

- 서명은 개인으로서 오픈스택 커밋터가 되겠다는 것을 클릭하면 되며, 클릭하고 동의하는 문구까지 작성하면 이제 Gerrit을 이용하기 위한 1차적인 설정은 완료됩니다.

.. image:: https://miro.medium.com/max/441/1*Wr8X2dBmmZ87xcAehWI90w.png
   :width: 1000px

- 참고로 위의 서명을 하는 페이지가 프로필 페이지인데, 본인의 Username도 꼭 해주세요.

- 추후에 터미널로 Gerrit 설정할 때 본인의 Gerrit Username을 반드시 등록해야하기 때문에 해당 설정을 안하고 진행하면, 에러가 발생할 수 있습니

1–2) SSH 키 생성하기
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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

2. Gerrit & Storyboard사용방법 안내
------------------------------------------------

2–1) Gerrit을 로컬에서 사용하기 위한 git-review 설치하기
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- 먼저 맥에서 당신이 Gerrit을 사용하는 상황이라면 아래 명령어를 사용해서 git-review라는 모듈을 설치해줍니다.

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


2–2) Storyboard 사용하기
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- 아마 Gerrit을 가입하면서 StoryBoard는 당연히 소셜 로그인 연동이 될테니 가입 절차는 생략하겠습니다.

- storyboard를 들어오면 프로젝트 검색 창에 sandbox라는 것이 있습니다 (하단 reference 참고).

- Storyboard는 전 세계 모든 OpenStack 사용자들이 OpenStack을 사용하면서 발생한 버그 등의 이슈를 올려놓는 곳인데, 그 중에 Sandbox라는 프로젝트는 Gerrit을 이용한 코드리뷰 테스트를 작업하기 위해 생성한 레포입니다.

- (그래서 모래 상자라고 해서 엉망으로 사용해도 문제가 안되니 프로젝트 이름을 지었나 유추해봅니다.)

.. image:: https://miro.medium.com/max/700/1*DnBa5gLkYBizSlfzyt02oA.png
   :width: 1000px

- 샌드박스 스토리 보드를 확인하고, "add stroy"를 클릭합니다.

.. image:: https://miro.medium.com/max/700/1*hD8brKFKja1GeLioYbcRpg.png
   :width: 1000px

- 당신이 sandbox 계정을 생성했다면 이제 sandbox에 첫번째 이슈를 생성해줍니다.

- Add Story를 눌러서 이슈 제목과 내용을 마음껏 작성하고, Save Changes를 클릭해주면 첫번째 스토리 보드가 생성되는 것을 볼 수 있습니다.

.. image:: https://miro.medium.com/max/700/1*UbDL-JSmH_fcCYJE5XvGXQ.png
   :width: 1000px

- 그럼 이슈가 정상적으로 등록되고 여기서 2가지가 확인 가능합니다.

    - Story ID : URL의 뒷부분 번호 확인
    - Task ID : task 하단의 5자리 번호 확인

- 이 2가지를 이제 커밋할 때 반드시 기록해야합니다.

3. git review를 이용한 커밋 메세지 작성 방법 및 푸시 방법 안내
---------------------------------------------------------------------------------------------

- 그리고 Sandbox를 사용하고 싶다면 프로젝트를 클론을 해줘야 합니다 (하단 reference 참고).

- 위의 링크를 클론해서 당신의 프로젝트 폴더에 넣고 해당 폴더의 경로로 들어가서 아무 파일이나 생성합니다.

- 저는 brain.py라고 파일을 생성후 print 문을 작성했습니다.

.. image:: https://miro.medium.com/max/1400/1*zdjP3J74Y7RI9cl1LfJRGQ.png
   :width: 1000px

- 파일들 명칭만 봐도 알겠지만 모두 테스틀 위해 대충 만든 레포라는 것을 볼 수 있습니다.

- 그리고 아래 명령어를 작성하면, 커밋 메세지 작업 규칙이 나와 있습니다.

.. code-block:: console

    > git add brain.py
    > git commit

- 작업 규칙은 제목 및 본문은 한 줄에 79자 내외, 그리고 storyboard & task ID를 하단에 작성해주는 것입니다.

.. image:: https://miro.medium.com/max/555/1*oQJliQaHshpeduLJTiBW7Q.png
   :width: 1000px


- 위의 이미지처럼 규칙을 지켜서 작성 후에 :wq를 입력하고 저장해줍니다.

- 그리고 git push origin <원본 레포>가 아닙니다.

- git review를 입력해주면 코드가 자동 push 되는 것을 볼 수 있습니다.

.. image:: https://miro.medium.com/max/700/1*C82ZJ0mhE0_Unq9cMUsrMg.png
   :width: 1000px

4. Gerrit을 사용한 코드 리뷰 남기기 및 수정 코드 커밋
----------------------------------------------------------------------------------

- 아래 이미지는 위의 링크를 따라 접속하면 볼 수 있는 Gerrit 코드 리뷰 페이지 입니다.

.. image:: https://miro.medium.com/max/1400/1*URKkXK0JFdGmK53PgwbSzg.png
   :width: 1000px


- 그럼 위의 페이지에서 제가 수정한 파일을 들어갈 수 있고, 거기서 키보드의 c를 누르면 누군가의 코드에 대해 리뷰를 남길 수 있습니다.

.. image:: https://miro.medium.com/max/700/1*hoRyr1U_J6B_zqrNMz9wtw.png
   :width: 1000px

- (참고로 본인이 커밋한 코드에 대해서는 셀프 코드 리뷰가 불가능합니다.)

- 아마 누군가 리뷰를 남겨줬다면 Code-Review 점수가 올라가 있을 것입니다.

.. image:: https://miro.medium.com/max/544/1*0NqcMl895GW8GOi0MnURpw.png
   :width: 1000px

- 처음 올린 코드 커밋에 대해 확인이 필요하면 리뷰를 남기고 0점을 주고 (아직 완성 못했으므로) 수정했다면 리뷰 점수를 일반 사용자는 2점까지 줄 수 있습니다.

.. image:: https://miro.medium.com/max/536/1*qeL-2ZW93davU_ScaMU0mg.png
   :width: 1000px


- 그리고 최종 merge를 하기 위해서는 workflow라는 점수가 있는데 이는 프로젝트를 매니징하거나 관리하는 분들이 줄 수 있기 때문에 일반 개발자가 부여할 수 없는 점수라고 보면 됩니다.

.. image:: https://miro.medium.com/max/700/1*vHnJguEsNDKIv8lijKCLxA.png
   :width: 1000px

- 그리고 -2, -1 점도 있는데 이건 코드에 심각한 결함이 있거나(-1점) 관리자가 해당 개발을 거부하는 경우 (-2점에 해당하며 코드를 잘못짜서 주는 점수는 아니라고 한다)라고 보면 됩니다.

- (당연히 위의 경우 merge는 될 수 없습니다.)

- 누군가 코드 리뷰를 남겨주면, 그 코드 리뷰에 따라 내가 작업한 코드를 수정해주고, 다시

.. code-block:: console

    > git add brain.py
    > git commit --ammend

- 를 작성해서 기존 커밋을 수정해서 다시 git review를 해줍니다.

- (기존 파일 수정본에 대해서는 새롭게 커밋하는 게 아니라서 해당 부분을 조심하셔야 합니다!)

5.Gerrit에서 코드가 Merge되는 방법
------------------------------------------------

.. image:: https://miro.medium.com/max/1400/1*BlpWVBfgEjbuK9mG68QXZg.png
   :width: 1000px

- 이렇게 되면 여기서 확대해서 볼 부분이 있는데

.. image:: https://miro.medium.com/max/648/1*idw5lTwc43BO18LmruZ4tg.png
   :width: 1000px

- 위의 코드 리뷰 화면 상단에 Base라고 되어 있는 부분을 Patchset1 로 변경하고 코드 수정을 완료하면 코드 리뷰 남겨준 사람에게 코드를 수정했다고 언급해주고 Done을 클릭합니다.

- (본인이이 커밋한 사람이 아니라 리뷰를 남긴 사람일 때 해당)

.. image:: https://miro.medium.com/max/700/1*Lxko2Bc0rolgsKLm7VTgyw.png
   :width: 1000px

- 그럼 리뷰를 남긴 사람의 글 오른편에 patchset2라고 되어 있는 것을 볼 수 있고 (처음 커밋에 리뷰를 남기면 patchset1이라 되어 있다) 최종 승인자가 workflow 점수를 주면 해당 코드가 merge되어 gerrit과 storyboard 모두 close 됩니다.

6. 에러 처리
------------------------------------------------

6–1) Remote rejected Error (git review 명령어 진행 시)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. image:: https://miro.medium.com/max/1400/1*EXRhwaIlcurUU6ISgmc20g.png
   :width: 1000px

- 팀원이 신규 커밋을 올려서 merge가 된 경우가 있었고, 새로운 커밋 히스토리가 생겼으므로 git pull을 해서 당신의 로컬 레포를 최신화시켜 줍니다.

- 만약 당신이 멋모르고 커밋을 먼저 했다면 (== 나) git reset — hard HEAD~1 라고 해서 당신 커밋기록을 지우고 다시 git pull을 해주면 됩니다.

6–2) Base → Patchset1로 변경 안하고 최종 커밋 커멘트 작성시 해결
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- 해당 커멘트를 수정하고, Base → Patchset1로 변경 후, Code Review & Workflow 점수를 부여합니다.

- (물론 실제 의미있는 코드에 대한 커밋이라면 Workflow 점수는 부여할 수 없습니다. 다만 현재는 테스트 상이니까 가능한 일입니다!)

- 그러면 Zuul이 알아서 CI를 돌면서 Storyboard & Gerrit 모두 자동 merge를 진행하게 됩니다.

7. 후기
------------------------------------------------

- 어려운 내용은 아닌데 손이 많이 가게 됩니다.

- 그리고 솔직히 커밋 관리가 일반적인 git 사용하는 상황보다 훨씬 간결하고, 한 번 올라간 커밋에 대해서는 amend를 사용해서 수정하는 방식으로 작업하는 게 마음에 듭니다.

- 아마 현재는 익숙하지 않기 때문에 여러가지로 절차들을 이해하는 과정이 쉽지는 않지만, 시간 문제라고 봅니다.

- merge를 하는 과정도 Zuul이라는 소프트웨어가 2가지 CI를 실행하는데 하나는 코드가 커밋되었을 때, 그리고 커밋한 코드를 전체 OpenStack 코드에 CI가 돌면서 문제는 없는지 실행하는 과정입니다.

- 방대한 프로젝트인만큼 여러가지로 잘못된 코드가 올라오는 과정을 많이 점검하는 것 같습니다.

- 새로운 툴과 버전 컨트롤을 하는 방식을 배우는 게 정말 즐겁습니다.

8.Reference
------------------------------------------------

- `Gerrit 컨트리뷰터 서약하기 <https://review.opendev.org/settings/#Agreements>`_

- `Storyboard Sandbox <https://storyboard.openstack.org/#!/project/opendev/sandbox>`_

- `Sandbox Repo <https://opendev.org/opendev/sandbox>`_

- `Gerrit 설정하기 국문 공식 문서 <https://docs.openstack.org/contributors/ko_KR/common/setup-gerrit.html>`_

- `Git Set up <https://docs.openstack.org/contributors/common/git.html#id1>`_

- `Using Gerrit <https://docs.openstack.org/contributors/code-and-documentation/using-gerrit.html>`_

- `OpenStack Code Review 방법 <https://docs.openstack.org/project-team-guide/review-the-openstack-way.html>`_

- `스토리보드 공식 설명 문서 <https://docs.openstack.org/infra/storyboard/>`_
