.. _fixture:
.. _fixtures:
.. _`fixture functions`:

pytest fixtures: 显式，模块化，可扩展
========================================================

.. currentmodule:: _pytest.python



.. _`xUnit`: http://en.wikipedia.org/wiki/XUnit
.. _`purpose of test fixtures`: http://en.wikipedia.org/wiki/Test_fixture#Software
.. _`依赖注入`: http://en.wikipedia.org/wiki/Dependency_injection

`测试fixtures` 的目的是提供一个固定基线，在此基础上测试可以可靠地重复执行。pytest
fixtures较经典xUnit风格的setup/teardown功能有了显著改进:

* fixtures具有显式的名称，并可以从声明它的测试函数，模块，类或整个项目中激活它。

* fixture以模块化的方式实现，因为每个fixture名称都触发一个 *fixture函数*，该函数本身可以使用其他fixture。

* fixture管理从简单的单元扩展到复杂的功能测试，允许根据配置和组件选项对fixture和测试进行参数化，
  或者在函数、类、模块及整个测试会话范围内重用fixtures。

此外，pytest继续支持 :ref:`经典xunit setup <xunitsetup>` 。您可以混合使用这两种样式，根据您的喜好从经典样式逐渐过渡到新样式。
您也可以从现有的 :ref:`unittest.TestCase样式 <unittest.TestCase>` 或 :ref:`基于nose <nosestyle>` 开始。


.. _`funcargs`:
.. _`funcarg mechanism`:
.. _`fixture function`:
.. _`@pytest.fixture`:
.. _`pytest.fixture`:

Fixtures作为函数参数
-----------------------------------------

测试函数通过将入参命名为fixture对象名来接收它们。对于每个参数，具有该名称的fixture函数会提供fixture对象。
Fixture函数通过标记 :py:func:`@pytest.fixture <_pytest.python.fixture>` 来注册。
让我们来看一个包含fixture和测试函数的简单测试模块::

    # content of ./test_smtpsimple.py
    import pytest

    @pytest.fixture
    def smtp_connection():
        import smtplib
        return smtplib.SMTP("smtp.gmail.com", 587, timeout=5)

    def test_ehlo(smtp_connection):
        response, msg = smtp_connection.ehlo()
        assert response == 250
        assert 0 # for demo purposes

这里， ``test_ehlo`` 需要 ``smtp_connection`` 的fixture值。pytest会发现并调用带有
:py:func:`@pytest.fixture <_pytest.python.fixture>` 标记的fixture函数 ``smtp_connection`` 。运行测试如下所示：

.. code-block:: pytest

    $ pytest test_smtpsimple.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 1 item

    test_smtpsimple.py F                                                 [100%]

    ================================= FAILURES =================================
    ________________________________ test_ehlo _________________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_ehlo(smtp_connection):
            response, msg = smtp_connection.ehlo()
            assert response == 250
    >       assert 0 # for demo purposes
    E       assert 0

    test_smtpsimple.py:11: AssertionError
    ========================= 1 failed in 0.12 seconds =========================

在错误回溯中，我们看到测试函数调用了 ``smtp_connection`` 参数，其 ``smtplib.SMTP()`` 实例由fixture函数创建。
测试函数因为我们故意的 ``assert 0`` 而失败。以下是 ``pytest`` 调用测试函数的具体流程：



1. pytest根据 ``test_`` 前缀 :ref:`发现 <test discovery>` ``test_ehlo`` 。测试函数需要一个名为
   ``smtp_connection`` 的入参，通过查找以fixture标记的 ``smtp_connection`` 函数，可以发现匹配的fixture函数。

2. 调用 ``smtp_connection()`` 创建实例。


3. 调用 ``test_ehlo(<smtp_connection instance>)`` ，然后在测试函数的最后一行失败了。

注意，如果您拼错了函数参数，或者希望使用不可用的参数，您将看到一个带有可用函数参数列表的错误。

.. note::

    你可以像这样：

    .. code-block:: bash

        pytest --fixtures test_simplefactory.py

    查看可用的fixtures（只有添加 ``-v`` 选项，才能显示以 ``_`` 开头的fixture）

Fixtures: 依赖注入的典型
---------------------------------------------------

Fixtures使得测试函数可以轻松地接受和处理特定的预初始化应用程序对象，而不必关心导入/设置/清理的细节。
这是一个 `依赖注入`_ 的典型例子，其中fixtures函数充当 *注入器* 的角色，而测试函数是fixture对象的 *消费者* 。

.. _`conftest.py`:
.. _`conftest`:

``conftest.py``: 共享fixture函数
------------------------------------------

如果在实现测试期间，您希望在多个测试文件中使用同一个fixture函数，您可以将它移动到 ``conftest.py`` 文件中。
您不需要在测试文件中导入fixture，它会自动被pytest发现。fixture函数的发现从测试类开始，然后是测试模块，
接着是 ``conftest.py`` 文件，最后是内置和第三方插件。

您还可以使用 ``conftest.py`` 文件来实现 :ref:`本地目录插件 <conftest.py plugins>` 。

共享测试数据
-----------------

如果您想让测试用例从文件中获取测试数据，那么将这些数据加载到fixture中不失为一个好主意。这利用了pytest的自动缓存机制。

另一个方法是在 ``tests`` 文件夹中添加测试数据。此外，社区插件也可用于管理这方面的测试，例如：
`pytest-datadir <https://pypi.org/project/pytest-datadir/>`__ 和
`pytest-datafiles <https://pypi.org/project/pytest-datafiles/>`__ 。

.. _smtpshared:

Scope：在测试类、模块或会话间共享fixture实例
----------------------------------------------------------------------------

.. regendoc:wipe

需要网络访问的fixture依赖于连接性，而且通常创建非常耗时。扩展之前的例子，我们可以向
:py:func:`@pytest.fixture <_pytest.python.fixture>` 添加 ``scope="module"`` 参数，使得fixture函数
``smtp_connection`` 只被每个测试 *模块* 调用一次（默认是每个测试 *函数* 调用一次）。
因此，测试模块中的多个测试函数将接收同一个fixture实例 ``smtp_connection`` 。对于 ``scope`` 可能的值有：
``function``, ``class``, ``module``, ``package`` 或 ``session`` 。

下一个例子将fixture函数放入单独的 ``conftest.py`` 文件中，所以目录内多个测试模块中的测试用例都可以访问fixture函数::

    # content of conftest.py
    import pytest
    import smtplib

    @pytest.fixture(scope="module")
    def smtp_connection():
        return smtplib.SMTP("smtp.gmail.com", 587, timeout=5)

fixture的名字仍是 ``smtp_connection`` ，您可以在任何测试或fixture函数中（ ``conftest.py`` 所在目录中或目录下）将
``smtp_connection`` 作为入参来访问它的结果::

    # content of test_module.py

    def test_ehlo(smtp_connection):
        response, msg = smtp_connection.ehlo()
        assert response == 250
        assert b"smtp.gmail.com" in msg
        assert 0  # for demo purposes

    def test_noop(smtp_connection):
        response, msg = smtp_connection.noop()
        assert response == 250
        assert 0  # for demo purposes

我们故意插入失败的 ``assert 0`` 语句以便检查发生了什么，现在运行测试用例：

.. code-block:: pytest

    $ pytest test_module.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 2 items

    test_module.py FF                                                    [100%]

    ================================= FAILURES =================================
    ________________________________ test_ehlo _________________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_ehlo(smtp_connection):
            response, msg = smtp_connection.ehlo()
            assert response == 250
            assert b"smtp.gmail.com" in msg
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:6: AssertionError
    ________________________________ test_noop _________________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_noop(smtp_connection):
            response, msg = smtp_connection.noop()
            assert response == 250
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:11: AssertionError
    ========================= 2 failed in 0.12 seconds =========================

您可以看到两个 ``assert 0`` 失败了，更重要的是您还可以看到相同的（模块范围） ``smtp_connection``
对象被传入两个测试函数中，因为pytest在回溯中显示入参值。因此，使用 ``smtp_connection`` 的两个测试函数运行速度和单个测试函数一样快，
因为它们重用了相同的实例。

如果您决定要使用会话范围的 ``smtp_connection`` 实例，只需要声明如下：

.. code-block:: python

    @pytest.fixture(scope="session")
    def smtp_connection():
        # the returned fixture value will be shared for
        # all tests needing it
        ...

最后， ``class`` 范围的fixture将会在每个测试 *类* 中被调用一次。

.. note::

    Pytest每次只缓存一个fixture实例。
    这意味着当使用参数化的fixture时，pytest可以在指定范围内多次调用fixture。


``package`` 范围 (实验性)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


在pytest 3.7中，引入了 ``package`` 范围。当测试用例的最后一个 *包* 完成时，包范围的fixtures终止。

.. warning::
    该功能被认为是 **实验性** 的，如果在更多使用该功能后，发现了隐藏情况或严重问题，则该功能可能在未来版本中被删除。

    谨慎使用此新功能，请务必报告您发现的任何问题。



顺序：首先实例化范围更大的fixture
----------------------------------------------------


在请求功能函数时，首先实例化范围更大的fixture（例如 ``session`` ），而不是范围较小的fixtures（例如 ``function`` 或 ``class`` ）。
具有相同范围的fixture，其相对顺序遵循测试函数中的声明顺序和fixtures间的依赖关系。Autouse
的fixture将会在显式使用fixture前被实例化。

请考虑如下代码：

.. literalinclude:: example/fixtures/test_fixtures_order.py

``test_foo`` 请求的fixtures，将按照以下顺序实例化：

1. ``s1``: 是最大范围的fixture (``session``)。
2. ``m1``: 是范围第二大的fixture (``module``)。
3. ``a1``: 是 ``function`` 范围的带有 ``autouse`` 的fixture，它将比同一范围内的其他fixtures优先实例化。
4. ``f3``: 是 ``function`` 范围的fixture，被 ``f1`` 依赖，此时需要实例化它。
5. ``f1``: 是 ``test_foo`` 参数列表中的第一个 ``function`` 范围的fixture。
6. ``f2``: 是 ``test_foo`` 参数列表中的最后一个 ``function`` 范围的fixture。


.. _`finalization`:

Fixture终止/执行teardown代码
-------------------------------------------------------------

pytest支持fixture在超出范围时执行特定的终结代码。通过使用 ``yield`` 语句代替 ``return`` ，则
*yield* 之后的所有代码都作为teardown代码：

.. code-block:: python

    # content of conftest.py

    import smtplib
    import pytest


    @pytest.fixture(scope="module")
    def smtp_connection():
        smtp_connection = smtplib.SMTP("smtp.gmail.com", 587, timeout=5)
        yield smtp_connection  # provide the fixture value
        print("teardown smtp")
        smtp_connection.close()

不管测试有任何异常状态， ``print`` 和 ``smtp.close()`` 语句都将在该模块的最后一个测试用例完成时执行。

让我们执行它：

.. code-block:: pytest

    $ pytest -s -q --tb=no
    FFteardown smtp

    2 failed in 0.12 seconds

我们看到 ``smtp_connection`` 实例是在两个测试用例完成后结束的。注意，如果我们用 ``scope='function'``
装饰fixture函数，那么每个测试用例前后都会进行fixture的设置和清理。无论在哪种情况下，
测试模块自身都无需修改或了解fixture设置的细节。

注意，我们还可以无缝地将 ``yield`` 语法和 ``with`` 语句结合使用。

.. code-block:: python

    # content of test_yield2.py

    import smtplib
    import pytest


    @pytest.fixture(scope="module")
    def smtp_connection():
        with smtplib.SMTP("smtp.gmail.com", 587, timeout=5) as smtp_connection:
            yield smtp_connection  # provide the fixture value


``smtp_connection`` 连接将在测试执行完毕后关闭，因为当 ``with`` 语句结束时， ``smtp_connection``
对象自动关闭。

无论fixture的 *setup* 代码是否引发异常，都将始终调用contextlib.ExitStack上下文管理终结器。
这对于正确关闭所有由fixture创建的资源非常方便，即使其中某个资源的创建/获取失败:


.. code-block:: python

    # content of test_yield3.py

    import contextlib

    import pytest

    from .utils import connect


    @pytest.fixture
    def equipments():
        with contextlib.ExitStack() as stack:
            yield [stack.enter_context(connect(port)) for port in ("C1", "C3", "C28")]

在上面的例子中，即使 ``"C28"`` 异常失败， ``"C1"`` 和 ``"C3"`` 仍会正确关闭。

注意，如果在 *setup* 代码（在 ``yield`` 关键字之前）期间发生异常，则不会调用 *teardown* 代码（在 ``yield`` 关键字之后）。

另一个替代方案是使用 `request-context`_ 对象的 ``addfinalizer`` 方法去注册终结函数，用来执行 *teardown* 代码。

下面是 ``smtp_connection`` fixture使用 ``addfinalizer`` 进行清理：

.. code-block:: python

    # content of conftest.py
    import smtplib
    import pytest


    @pytest.fixture(scope="module")
    def smtp_connection(request):
        smtp_connection = smtplib.SMTP("smtp.gmail.com", 587, timeout=5)

        def fin():
            print("teardown smtp_connection")
            smtp_connection.close()

        request.addfinalizer(fin)
        return smtp_connection  # provide the fixture value


下面是 ``equipments`` fixture使用 ``addfinalizer`` 进行清理：

.. code-block:: python

    # content of test_yield3.py

    import contextlib
    import functools

    import pytest

    from .utils import connect


    @pytest.fixture
    def equipments(request):
        r = []
        for port in ("C1", "C3", "C28"):
            cm = connect(port)
            equip = cm.__enter__()
            request.addfinalizer(functools.partial(cm.__exit__, None, None, None))
            r.append(equip)
        return r


``yield`` 和 ``addfinalizer`` 的工作原理类似，都是在测试结束后调用它们的代码。当然，如果在终止函数注册之前发生异常，
则不会执行它。


.. _`request-context`:

Fixtures能内省请求的测试上下文
-------------------------------------------------------------

Fixture函数可以接受 :py:class:`request <FixtureRequest>` 对象来内省"请求"的测试函数，类或模块的上下文。
进一步扩展前面的 ``smtp_connection`` fixture例子，让我们从使用该fixture的测试模块中读取一个可选的服务器URL::

    # content of conftest.py
    import pytest
    import smtplib

    @pytest.fixture(scope="module")
    def smtp_connection(request):
        server = getattr(request.module, "smtpserver", "smtp.gmail.com")
        smtp_connection = smtplib.SMTP(server, 587, timeout=5)
        yield smtp_connection
        print("finalizing %s (%s)" % (smtp_connection, server))
        smtp_connection.close()

我们使用 ``request.module`` 属性从测试模块中获取 ``smtpserver`` 可选的属性。如果我们再次执行，没有太大变化：

.. code-block:: pytest

    $ pytest -s -q --tb=no
    FFfinalizing <smtplib.SMTP object at 0xdeadbeef> (smtp.gmail.com)

    2 failed in 0.12 seconds

让我们快速创建另外一个测试模块，它在自己的模块命名空间中设置服务器URL::

    # content of test_anothersmtp.py

    smtpserver = "mail.python.org"  # will be read by smtp fixture

    def test_showhelo(smtp_connection):
        assert 0, smtp_connection.helo()

执行它：

.. code-block:: pytest

    $ pytest -qq --tb=short test_anothersmtp.py
    F                                                                    [100%]
    ================================= FAILURES =================================
    ______________________________ test_showhelo _______________________________
    test_anothersmtp.py:5: in test_showhelo
        assert 0, smtp_connection.helo()
    E   AssertionError: (250, b'mail.python.org')
    E   assert 0
    ------------------------- Captured stdout teardown -------------------------
    finalizing <smtplib.SMTP object at 0xdeadbeef> (mail.python.org)

瞧！ ``smtp_connection`` fixture函数从模块命名空间中获取到邮件服务器名称。

.. _`fixture-factory`:

工厂fixtures
-------------------------------------------------------------

"工厂fixture"模式有助于单个测试用例多次使用fixture的场景。fixture不是直接返回数据，
而是返回一个生成数据的函数。该函数可以在测试用例中被调用多次。

工厂可以有参数::

    @pytest.fixture
    def make_customer_record():

        def _make_customer_record(name):
            return {
                "name": name,
                "orders": []
            }

        return _make_customer_record


    def test_customer_records(make_customer_record):
        customer_1 = make_customer_record("Lisa")
        customer_2 = make_customer_record("Mike")
        customer_3 = make_customer_record("Meredith")

如果需要管理工厂创建的数据，fixture可以处理::

    @pytest.fixture
    def make_customer_record():

        created_records = []

        def _make_customer_record(name):
            record = models.Customer(name=name, orders=[])
            created_records.append(record)
            return record

        yield _make_customer_record

        for record in created_records:
            record.destroy()


    def test_customer_records(make_customer_record):
        customer_1 = make_customer_record("Lisa")
        customer_2 = make_customer_record("Mike")
        customer_3 = make_customer_record("Meredith")


.. _`fixture-parametrize`:

参数化fixtures
-----------------------------------------------------------------

参数化的fixture函数可以被多次调用，每次执行一组依赖测试，即依赖于这个fixture的测试。
测试函数通常不需要注意重新运行的情况。Fixture参数化可以通过多种方式配置，有助于为组件编写详尽的功能测试。

扩展之前的例子，我们可以标记fixture来创建两个 ``smtp_connection`` fixture实例，这将导致所有使用该fixture
的测试用例都运行两次。fixture函数通过特殊的 :py:class:`request <FixtureRequest>` 对象访问每个参数::

    # content of conftest.py
    import pytest
    import smtplib

    @pytest.fixture(scope="module",
                    params=["smtp.gmail.com", "mail.python.org"])
    def smtp_connection(request):
        smtp_connection = smtplib.SMTP(request.param, 587, timeout=5)
        yield smtp_connection
        print("finalizing %s" % smtp_connection)
        smtp_connection.close()

主要的变化是使用 :py:func:`@pytest.fixture <_pytest.python.fixture>` 声明 ``params`` ，
fixture函数的取值列表都将会执行，并可以通过  ``request.param`` 访问某个取值。不需要更改任何测试函数代码，我们再来一次:

.. code-block:: pytest

    $ pytest -q test_module.py
    FFFF                                                                 [100%]
    ================================= FAILURES =================================
    ________________________ test_ehlo[smtp.gmail.com] _________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_ehlo(smtp_connection):
            response, msg = smtp_connection.ehlo()
            assert response == 250
            assert b"smtp.gmail.com" in msg
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:6: AssertionError
    ________________________ test_noop[smtp.gmail.com] _________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_noop(smtp_connection):
            response, msg = smtp_connection.noop()
            assert response == 250
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:11: AssertionError
    ________________________ test_ehlo[mail.python.org] ________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_ehlo(smtp_connection):
            response, msg = smtp_connection.ehlo()
            assert response == 250
    >       assert b"smtp.gmail.com" in msg
    E       AssertionError: assert b'smtp.gmail.com' in b'mail.python.org\nPIPELINING\nSIZE 51200000\nETRN\nSTARTTLS\nAUTH DIGEST-MD5 NTLM CRAM-MD5\nENHANCEDSTATUSCODES\n8BITMIME\nDSN\nSMTPUTF8\nCHUNKING'

    test_module.py:5: AssertionError
    -------------------------- Captured stdout setup ---------------------------
    finalizing <smtplib.SMTP object at 0xdeadbeef>
    ________________________ test_noop[mail.python.org] ________________________

    smtp_connection = <smtplib.SMTP object at 0xdeadbeef>

        def test_noop(smtp_connection):
            response, msg = smtp_connection.noop()
            assert response == 250
    >       assert 0  # for demo purposes
    E       assert 0

    test_module.py:11: AssertionError
    ------------------------- Captured stdout teardown -------------------------
    finalizing <smtplib.SMTP object at 0xdeadbeef>
    4 failed in 0.12 seconds

我们看到两个测试函数分别针对不同的 ``smtp_connection`` 实例运行了两次。还要注意，使用
``mail.python.org`` 连接， ``test_ehlo`` 的第二次测试失败了，因为预期的服务器字符串与实际的不一致。

pytest会为每个参数化fixture中的fixture值构建一个测试ID字符串，例如上面例子中的
``test_ehlo[smtp.gmail.com]`` 和 ``test_ehlo[mail.python.org]`` 。这些ID可以与 ``-k`` 结合使用，用于选择特定的用例来运行。
pytest带有 ``--collect-only`` 运行时，会显示生成的ID。

数字、字符串、布尔值和 None 将在测试ID中使用它们通常的字符串表示形式。
对于其他对象，pytest将基于参数名创建一个字符串。可以使用 ``ids`` 关键字参数为某个fixture值定制测试ID中使用的字符串::

   # content of test_ids.py
   import pytest

   @pytest.fixture(params=[0, 1], ids=["spam", "ham"])
   def a(request):
       return request.param

   def test_a(a):
       pass

   def idfn(fixture_value):
       if fixture_value == 0:
           return "eggs"
       else:
           return None

   @pytest.fixture(params=[0, 1], ids=idfn)
   def b(request):
       return request.param

   def test_b(b):
       pass

上面展示了 ``ids`` 可以是要使用的字符串列表，也可以是使用fixture值并返回字符串的函数。
在后一种情况下，如果函数返回 ``None`` ，则使用pytest自动生成的ID。

运行以上测试，得到测试ID如下:

.. code-block:: pytest

   $ pytest --collect-only
   =========================== test session starts ============================
   platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
   cachedir: $PYTHON_PREFIX/.pytest_cache
   rootdir: $REGENDOC_TMPDIR
   collected 10 items
   <Module test_anothersmtp.py>
     <Function test_showhelo[smtp.gmail.com]>
     <Function test_showhelo[mail.python.org]>
   <Module test_ids.py>
     <Function test_a[spam]>
     <Function test_a[ham]>
     <Function test_b[eggs]>
     <Function test_b[1]>
   <Module test_module.py>
     <Function test_ehlo[smtp.gmail.com]>
     <Function test_noop[smtp.gmail.com]>
     <Function test_ehlo[mail.python.org]>
     <Function test_noop[mail.python.org]>

   ======================= no tests ran in 0.12 seconds =======================

.. _`fixture-parametrize-marks`:

Using marks with parametrized fixtures
--------------------------------------

:func:`pytest.param` can be used to apply marks in values sets of parametrized fixtures in the same way
that they can be used with :ref:`@pytest.mark.parametrize <@pytest.mark.parametrize>`.

Example::

    # content of test_fixture_marks.py
    import pytest
    @pytest.fixture(params=[0, 1, pytest.param(2, marks=pytest.mark.skip)])
    def data_set(request):
        return request.param

    def test_data(data_set):
        pass

Running this test will *skip* the invocation of ``data_set`` with value ``2``:

.. code-block:: pytest

    $ pytest test_fixture_marks.py -v
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 3 items

    test_fixture_marks.py::test_data[0] PASSED                           [ 33%]
    test_fixture_marks.py::test_data[1] PASSED                           [ 66%]
    test_fixture_marks.py::test_data[2] SKIPPED                          [100%]

    =================== 2 passed, 1 skipped in 0.12 seconds ====================

.. _`interdependent fixtures`:

Modularity: using fixtures from a fixture function
----------------------------------------------------------

You can not only use fixtures in test functions but fixture functions
can use other fixtures themselves.  This contributes to a modular design
of your fixtures and allows re-use of framework-specific fixtures across
many projects.  As a simple example, we can extend the previous example
and instantiate an object ``app`` where we stick the already defined
``smtp_connection`` resource into it::

    # content of test_appsetup.py

    import pytest

    class App(object):
        def __init__(self, smtp_connection):
            self.smtp_connection = smtp_connection

    @pytest.fixture(scope="module")
    def app(smtp_connection):
        return App(smtp_connection)

    def test_smtp_connection_exists(app):
        assert app.smtp_connection

Here we declare an ``app`` fixture which receives the previously defined
``smtp_connection`` fixture and instantiates an ``App`` object with it.  Let's run it:

.. code-block:: pytest

    $ pytest -v test_appsetup.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 2 items

    test_appsetup.py::test_smtp_connection_exists[smtp.gmail.com] PASSED [ 50%]
    test_appsetup.py::test_smtp_connection_exists[mail.python.org] PASSED [100%]

    ========================= 2 passed in 0.12 seconds =========================

Due to the parametrization of ``smtp_connection``, the test will run twice with two
different ``App`` instances and respective smtp servers.  There is no
need for the ``app`` fixture to be aware of the ``smtp_connection``
parametrization because pytest will fully analyse the fixture dependency graph.

Note that the ``app`` fixture has a scope of ``module`` and uses a
module-scoped ``smtp_connection`` fixture.  The example would still work if
``smtp_connection`` was cached on a ``session`` scope: it is fine for fixtures to use
"broader" scoped fixtures but not the other way round:
A session-scoped fixture could not use a module-scoped one in a
meaningful way.


.. _`automatic per-resource grouping`:

Automatic grouping of tests by fixture instances
----------------------------------------------------------

.. regendoc: wipe

pytest minimizes the number of active fixtures during test runs.
If you have a parametrized fixture, then all the tests using it will
first execute with one instance and then finalizers are called
before the next fixture instance is created.  Among other things,
this eases testing of applications which create and use global state.

The following example uses two parametrized fixtures, one of which is
scoped on a per-module basis, and all the functions perform ``print`` calls
to show the setup/teardown flow::

    # content of test_module.py
    import pytest

    @pytest.fixture(scope="module", params=["mod1", "mod2"])
    def modarg(request):
        param = request.param
        print("  SETUP modarg %s" % param)
        yield param
        print("  TEARDOWN modarg %s" % param)

    @pytest.fixture(scope="function", params=[1,2])
    def otherarg(request):
        param = request.param
        print("  SETUP otherarg %s" % param)
        yield param
        print("  TEARDOWN otherarg %s" % param)

    def test_0(otherarg):
        print("  RUN test0 with otherarg %s" % otherarg)
    def test_1(modarg):
        print("  RUN test1 with modarg %s" % modarg)
    def test_2(otherarg, modarg):
        print("  RUN test2 with otherarg %s and modarg %s" % (otherarg, modarg))


Let's run the tests in verbose mode and with looking at the print-output:

.. code-block:: pytest

    $ pytest -v -s test_module.py
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y -- $PYTHON_PREFIX/bin/python
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collecting ... collected 8 items

    test_module.py::test_0[1]   SETUP otherarg 1
      RUN test0 with otherarg 1
    PASSED  TEARDOWN otherarg 1

    test_module.py::test_0[2]   SETUP otherarg 2
      RUN test0 with otherarg 2
    PASSED  TEARDOWN otherarg 2

    test_module.py::test_1[mod1]   SETUP modarg mod1
      RUN test1 with modarg mod1
    PASSED
    test_module.py::test_2[mod1-1]   SETUP otherarg 1
      RUN test2 with otherarg 1 and modarg mod1
    PASSED  TEARDOWN otherarg 1

    test_module.py::test_2[mod1-2]   SETUP otherarg 2
      RUN test2 with otherarg 2 and modarg mod1
    PASSED  TEARDOWN otherarg 2

    test_module.py::test_1[mod2]   TEARDOWN modarg mod1
      SETUP modarg mod2
      RUN test1 with modarg mod2
    PASSED
    test_module.py::test_2[mod2-1]   SETUP otherarg 1
      RUN test2 with otherarg 1 and modarg mod2
    PASSED  TEARDOWN otherarg 1

    test_module.py::test_2[mod2-2]   SETUP otherarg 2
      RUN test2 with otherarg 2 and modarg mod2
    PASSED  TEARDOWN otherarg 2
      TEARDOWN modarg mod2


    ========================= 8 passed in 0.12 seconds =========================

You can see that the parametrized module-scoped ``modarg`` resource caused an
ordering of test execution that lead to the fewest possible "active" resources.
The finalizer for the ``mod1`` parametrized resource was executed before the
``mod2`` resource was setup.

In particular notice that test_0 is completely independent and finishes first.
Then test_1 is executed with ``mod1``, then test_2 with ``mod1``, then test_1
with ``mod2`` and finally test_2 with ``mod2``.

The ``otherarg`` parametrized resource (having function scope) was set up before
and teared down after every test that used it.


.. _`usefixtures`:

Using fixtures from classes, modules or projects
----------------------------------------------------------------------

.. regendoc:wipe

Sometimes test functions do not directly need access to a fixture object.
For example, tests may require to operate with an empty directory as the
current working directory but otherwise do not care for the concrete
directory.  Here is how you can use the standard `tempfile
<http://docs.python.org/library/tempfile.html>`_ and pytest fixtures to
achieve it.  We separate the creation of the fixture into a conftest.py
file::

    # content of conftest.py

    import pytest
    import tempfile
    import os

    @pytest.fixture()
    def cleandir():
        newpath = tempfile.mkdtemp()
        os.chdir(newpath)

and declare its use in a test module via a ``usefixtures`` marker::

    # content of test_setenv.py
    import os
    import pytest

    @pytest.mark.usefixtures("cleandir")
    class TestDirectoryInit(object):
        def test_cwd_starts_empty(self):
            assert os.listdir(os.getcwd()) == []
            with open("myfile", "w") as f:
                f.write("hello")

        def test_cwd_again_starts_empty(self):
            assert os.listdir(os.getcwd()) == []

Due to the ``usefixtures`` marker, the ``cleandir`` fixture
will be required for the execution of each test method, just as if
you specified a "cleandir" function argument to each of them.  Let's run it
to verify our fixture is activated and the tests pass:

.. code-block:: pytest

    $ pytest -q
    ..                                                                   [100%]
    2 passed in 0.12 seconds

You can specify multiple fixtures like this:

.. code-block:: python

    @pytest.mark.usefixtures("cleandir", "anotherfixture")
    def test():
        ...

and you may specify fixture usage at the test module level, using
a generic feature of the mark mechanism:

.. code-block:: python

    pytestmark = pytest.mark.usefixtures("cleandir")

Note that the assigned variable *must* be called ``pytestmark``, assigning e.g.
``foomark`` will not activate the fixtures.

It is also possible to put fixtures required by all tests in your project
into an ini-file:

.. code-block:: ini

    # content of pytest.ini
    [pytest]
    usefixtures = cleandir


.. warning::

    Note this mark has no effect in **fixture functions**. For example,
    this **will not work as expected**:

    .. code-block:: python

        @pytest.mark.usefixtures("my_other_fixture")
        @pytest.fixture
        def my_fixture_that_sadly_wont_use_my_other_fixture():
            ...

    Currently this will not generate any error or warning, but this is intended
    to be handled by `#3664 <https://github.com/pytest-dev/pytest/issues/3664>`_.


.. _`autouse`:
.. _`autouse fixtures`:

Autouse fixtures (xUnit setup on steroids)
----------------------------------------------------------------------

.. regendoc:wipe

Occasionally, you may want to have fixtures get invoked automatically
without declaring a function argument explicitly or a `usefixtures`_ decorator.
As a practical example, suppose we have a database fixture which has a
begin/rollback/commit architecture and we want to automatically surround
each test method by a transaction and a rollback.  Here is a dummy
self-contained implementation of this idea::

    # content of test_db_transact.py

    import pytest

    class DB(object):
        def __init__(self):
            self.intransaction = []
        def begin(self, name):
            self.intransaction.append(name)
        def rollback(self):
            self.intransaction.pop()

    @pytest.fixture(scope="module")
    def db():
        return DB()

    class TestClass(object):
        @pytest.fixture(autouse=True)
        def transact(self, request, db):
            db.begin(request.function.__name__)
            yield
            db.rollback()

        def test_method1(self, db):
            assert db.intransaction == ["test_method1"]

        def test_method2(self, db):
            assert db.intransaction == ["test_method2"]

The class-level ``transact`` fixture is marked with *autouse=true*
which implies that all test methods in the class will use this fixture
without a need to state it in the test function signature or with a
class-level ``usefixtures`` decorator.

If we run it, we get two passing tests:

.. code-block:: pytest

    $ pytest -q
    ..                                                                   [100%]
    2 passed in 0.12 seconds

Here is how autouse fixtures work in other scopes:

- autouse fixtures obey the ``scope=`` keyword-argument: if an autouse fixture
  has ``scope='session'`` it will only be run once, no matter where it is
  defined. ``scope='class'`` means it will be run once per class, etc.

- if an autouse fixture is defined in a test module, all its test
  functions automatically use it.

- if an autouse fixture is defined in a conftest.py file then all tests in
  all test modules below its directory will invoke the fixture.

- lastly, and **please use that with care**: if you define an autouse
  fixture in a plugin, it will be invoked for all tests in all projects
  where the plugin is installed.  This can be useful if a fixture only
  anyway works in the presence of certain settings e. g. in the ini-file.  Such
  a global fixture should always quickly determine if it should do
  any work and avoid otherwise expensive imports or computation.

Note that the above ``transact`` fixture may very well be a fixture that
you want to make available in your project without having it generally
active.  The canonical way to do that is to put the transact definition
into a conftest.py file **without** using ``autouse``::

    # content of conftest.py
    @pytest.fixture
    def transact(request, db):
        db.begin()
        yield
        db.rollback()

and then e.g. have a TestClass using it by declaring the need::

    @pytest.mark.usefixtures("transact")
    class TestClass(object):
        def test_method1(self):
            ...

All test methods in this TestClass will use the transaction fixture while
other test classes or functions in the module will not use it unless
they also add a ``transact`` reference.

Overriding fixtures on various levels
-------------------------------------

In relatively large test suite, you most likely need to ``override`` a ``global`` or ``root`` fixture with a ``locally``
defined one, keeping the test code readable and maintainable.

Override a fixture on a folder (conftest) level
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Given the tests file structure is:

::

    tests/
        __init__.py

        conftest.py
            # content of tests/conftest.py
            import pytest

            @pytest.fixture
            def username():
                return 'username'

        test_something.py
            # content of tests/test_something.py
            def test_username(username):
                assert username == 'username'

        subfolder/
            __init__.py

            conftest.py
                # content of tests/subfolder/conftest.py
                import pytest

                @pytest.fixture
                def username(username):
                    return 'overridden-' + username

            test_something.py
                # content of tests/subfolder/test_something.py
                def test_username(username):
                    assert username == 'overridden-username'

As you can see, a fixture with the same name can be overridden for certain test folder level.
Note that the ``base`` or ``super`` fixture can be accessed from the ``overriding``
fixture easily - used in the example above.

Override a fixture on a test module level
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Given the tests file structure is:

::

    tests/
        __init__.py

        conftest.py
            # content of tests/conftest.py
            import pytest

            @pytest.fixture
            def username():
                return 'username'

        test_something.py
            # content of tests/test_something.py
            import pytest

            @pytest.fixture
            def username(username):
                return 'overridden-' + username

            def test_username(username):
                assert username == 'overridden-username'

        test_something_else.py
            # content of tests/test_something_else.py
            import pytest

            @pytest.fixture
            def username(username):
                return 'overridden-else-' + username

            def test_username(username):
                assert username == 'overridden-else-username'

In the example above, a fixture with the same name can be overridden for certain test module.


Override a fixture with direct test parametrization
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Given the tests file structure is:

::

    tests/
        __init__.py

        conftest.py
            # content of tests/conftest.py
            import pytest

            @pytest.fixture
            def username():
                return 'username'

            @pytest.fixture
            def other_username(username):
                return 'other-' + username

        test_something.py
            # content of tests/test_something.py
            import pytest

            @pytest.mark.parametrize('username', ['directly-overridden-username'])
            def test_username(username):
                assert username == 'directly-overridden-username'

            @pytest.mark.parametrize('username', ['directly-overridden-username-other'])
            def test_username_other(other_username):
                assert other_username == 'other-directly-overridden-username-other'

In the example above, a fixture value is overridden by the test parameter value. Note that the value of the fixture
can be overridden this way even if the test doesn't use it directly (doesn't mention it in the function prototype).


Override a parametrized fixture with non-parametrized one and vice versa
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Given the tests file structure is:

::

    tests/
        __init__.py

        conftest.py
            # content of tests/conftest.py
            import pytest

            @pytest.fixture(params=['one', 'two', 'three'])
            def parametrized_username(request):
                return request.param

            @pytest.fixture
            def non_parametrized_username(request):
                return 'username'

        test_something.py
            # content of tests/test_something.py
            import pytest

            @pytest.fixture
            def parametrized_username():
                return 'overridden-username'

            @pytest.fixture(params=['one', 'two', 'three'])
            def non_parametrized_username(request):
                return request.param

            def test_username(parametrized_username):
                assert parametrized_username == 'overridden-username'

            def test_parametrized_username(non_parametrized_username):
                assert non_parametrized_username in ['one', 'two', 'three']

        test_something_else.py
            # content of tests/test_something_else.py
            def test_username(parametrized_username):
                assert parametrized_username in ['one', 'two', 'three']

            def test_username(non_parametrized_username):
                assert non_parametrized_username == 'username'

In the example above, a parametrized fixture is overridden with a non-parametrized version, and
a non-parametrized fixture is overridden with a parametrized version for certain test module.
The same applies for the test folder level obviously.
