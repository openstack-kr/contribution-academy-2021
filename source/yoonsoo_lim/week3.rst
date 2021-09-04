
3주차 과제 - openstack server list 에 field 추가하기
=======================================================

1. server list의 결과값을 반환하는 함수를 찾아보자
---------------------------------------------------

    * 코드를 보며 함수를 계속 찾아보면

    .. code-block:: python

        File: openstackclient/shell.py
            145 return OpenStackShell().run(argv)

        File: osc_lib/shell.py
            134 ret_val = super(OpenStackShell, self).run(argv)

        File: cliff/app.py
            277 result = self.run_subcommand(remainder)

        File: cliff/app.py
            714 ret_value = super(OpenStackShell, self).run_subcommand(argv)

        File: cliff/app.py
            402 result = cmd.run(parsed_args)


    * 쭉 들어오다보면 server.py까지 오는데 여기에 2262~2330L보면 컬럼에 filed값들이 보인다.
    * 뭔가 여기에 느낌이 있다!

    .. code-block:: python

        2367 data = compute_client.servers.list(search_opts=search_opts,
        2368                                    marker=marker_id,
        2369                                    limit=parsed_args.limit)

    * 위를 실행시키고 파이참의 Debugger에서 data를 확인하면 아래처럼 값이 들어있다.

    .. image:: ./images/data_value.png

    * 이 값을 보기 전엔 나는 server list를 하면 보이는 필드값들만 추출하는줄 알았는데 아니였다.
      인스턴스의 모든 정보를 로딩해서 가지고 있었던거였다.
    * 그럼 이제 필드를 추가하고 해당되는 필드의 값을 넣을면 될거같다.
      그러면 어떤 방식으로 필드를 선정하고 값을 넣는지 찾아보자

    |

    * 맨밑에 2449L~2463L이 뭔가 있어보인다. utils.get_item_properties를
      또 들어가보자

    .. code-block:: python

        File: openstackclient/compute/v2/server.py
            2452 utils.get_item_properties(
            2453            s, columns,
            2454            mixed_case_fields=mixed_case_fields,
            2455            formatters={
            2456                'OS-EXT-STS:power_state': PowerStateColumn,
            2457                'Networks': format_columns.DictListColumn,
            2458                'Metadata': format_columns.DictColumn,
            2459            },
            2460        ) for s in data


    * 그리고 utils/__init__.py에 전달된 값들은 아래와 같다.

    .. image:: images/field.png

    |

    .. image:: images/value.png

    * 508L을 보면 getattr함수를 이용해 item에 field의 해당하는 값이 있으면
      data에 넣는것을 확인할 수 있다.
    * 그렇다면 field에 project id와 created at를 추가하면 될거같다!

    |

    * 다시 openstackclient/compute/v2/server.py의 2311L으로 다시 가자
    * 위에 if문들은 --long옵션과 --no-name-lookup옵션을 사용했을때 field값
      이므로 2311L의 field값을 수정하면 된다.

    |

    * columns는 아까 item에 들어있는 field이름이랑 맞춰서 넣어야한다.
    * column_headers는 출력할 field의 이름을 넣으면 된다.

    .. image:: images/columns.png

    |

    .. image:: images/column_headers.png


2. field 추가한 server list 출력
--------------------------------------

    .. image:: images/result.png