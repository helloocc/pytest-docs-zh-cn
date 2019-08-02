
Monkeypatching/mocking模块和环境
================================================================

.. currentmodule:: _pytest.monkeypatch

有时，测试需要调用依赖全局设置的功能，或者不方便测试的代码（如网络访问）。
使用 ``monkeypatch`` fixture可以安全地设置/删除属性、字典项和环境变量，或者修改导入的 ``sys.path`` 。


``monkeypatch`` fixture为测试中安全地修改和模拟功能测试提供了方法：

.. code-block:: python

    monkeypatch.setattr(obj, name, value, raising=True)
    monkeypatch.delattr(obj, name, raising=True)
    monkeypatch.setitem(mapping, name, value)
    monkeypatch.delitem(obj, name, raising=True)
    monkeypatch.setenv(name, value, prepend=False)
    monkeypatch.delenv(name, raising=True)
    monkeypatch.syspath_prepend(path)
    monkeypatch.chdir(path)

所有修改都将在请求的测试函数或fixture完成后撤销。 ``raising`` 参数决定了如果设置/删除操作的目标不存在时，
是否会引发 ``KeyError`` 或 ``AttributeError`` 。

考虑以下场景：

1. 为测试修改函数的行为或类的属性。例如，您不会对某个API调用或数据库连接进行测试，但知道预期的输出应该是什么。
   使用 :py:meth:`monkeypatch.setattr` 为您想要的测试行为来修补函数或属性，这可以包括您自己的函数。
   使用 :py:meth:`monkeypatch.delattr` 删除测试的函数或属性。

2. 修改字典的值。例如，您想为某些测试用例修改一个全局配置项。使用 :py:meth:`monkeypatch.setitem` 为测试修补字典，
   使用 :py:meth:`monkeypatch.delitem` 删除字典项。

3. 为测试修改环境变量。例如，测试缺少环境变量时，或已知变量设定多个值时的程序行为。
   可以对这些补丁使用 :py:meth:`monkeypatch.setenv` 和 :py:meth:`monkeypatch.delenv` 。

4. 使用 :py:meth:`monkeypatch.syspath_prepend` 安全地修改系统 ``$PATH`` ，使用 :py:meth:`monkeypatch.chdir`
   在测试期间更改当前工作目录的上下文。

参阅 `monkeypatch blog post`_ 查看介绍材料和讨论。

.. _`monkeypatch blog post`: http://tetamap.wordpress.com/2009/03/03/monkeypatching-in-unit-tests-done-right/

简单示例：monkeypatching函数
----------------------------------------

考虑一个使用用户目录的场景。在测试上下文中，您不希望测试依赖于正在运行的用户。
``monkeypatch`` 可以对依赖于用户的函数进行修补，始终返回特定值。

在该示例中， :py:meth:`monkeypatch.setattr` 用于修补 ``Path.home`` ，从而在运行测试时始终使用已知的测试路径 ``Path("/abc")`` 。
这消除了测试对于运行用户的依赖性。必须在使用补丁函数之前调用 :py:meth:`monkeypatch.setattr` ，
测试函数完成后， ``Path.home`` 的修改将被撤销。

.. code-block:: python

    # contents of test_module.py with source code and the test
    from pathlib import Path


    def getssh():
        """Simple function to return expanded homedir ssh path."""
        return Path.home() / ".ssh"


    def test_getssh(monkeypatch):
        # mocked return function to replace Path.home
        # always return '/abc'
        def mockreturn():
            return Path("/abc")

        # Application of the monkeypatch to replace Path.home
        # with the behavior of mockreturn defined above.
        monkeypatch.setattr(Path, "home", mockreturn)

        # Calling getssh() will use mockreturn in place of Path.home
        # for this test with the monkeypatch.
        x = getssh()
        assert x == Path("/abc/.ssh")

Monkeypatching返回对象：构建mock类
------------------------------------------------------

:py:meth:`monkeypatch.setattr` 可以与类一起使用，mock对象替代函数的返回值。
设想一个接收API url并返回json响应的简单函数。

.. code-block:: python

    # contents of app.py, a simple API retrieval example
    import requests


    def get_json(url):
        """Takes a URL, and returns the JSON."""
        r = requests.get(url)
        return r.json()

我们需要mock返回的响应对象 ``r`` 来进行测试，Mock的 ``r`` 需要一个 ``.json()`` 方法，该方法返回一个字典。
我们可以在测试文件中定义一个代表 ``r`` 的类。

.. code-block:: python

    # contents of test_app.py, a simple test for our API retrieval
    # import requests for the purposes of monkeypatching
    import requests

    # our app.py that includes the get_json() function
    # this is the previous code block example
    import app

    # custom class to be the mock return value
    # will override the requests.Response returned from requests.get
    class MockResponse:

        # mock json() method always returns a specific testing dictionary
        @staticmethod
        def json():
            return {"mock_key": "mock_response"}


    def test_get_json(monkeypatch):

        # Any arguments may be passed and mock_get() will always return our
        # mocked object, which only has the .json() method.
        def mock_get(*args, **kwargs):
            return MockResponse()

        # apply the monkeypatch for requests.get to mock_get
        monkeypatch.setattr(requests, "get", mock_get)

        # app.get_json, which contains requests.get, uses the monkeypatch
        result = app.get_json("https://fakeurl")
        assert result["mock_key"] == "mock_response"


``monkeypatch`` 通过 ``mock_get`` 函数对 ``requests.get`` 进行mock。
``mock_get`` 函数返回一个 ``MockResponse`` 类的实例，该类定义了一个返回已知测试字典的 ``json()`` 方法，且不需要连接任何外部的API。

您可以为正在测试的场景构建复杂度适当的 ``MockResponse`` 类。例如，它可以包含一个总是返回 ``True`` 的 ``ok`` 属性，
或者基于输入字符串从mock的 ``json()`` 方法中返回不同的值。

使用 ``fixture`` 可以在测试中共享这个mock。

.. code-block:: python

    # contents of test_app.py, a simple test for our API retrieval
    import pytest
    import requests

    # app.py that includes the get_json() function
    import app

    # custom class to be the mock return value of requests.get()
    class MockResponse:
        @staticmethod
        def json():
            return {"mock_key": "mock_response"}


    # monkeypatched requests.get moved to a fixture
    @pytest.fixture
    def mock_response(monkeypatch):
        """Requests.get() mocked to return {'mock_key':'mock_response'}."""

        def mock_get(*args, **kwargs):
            return MockResponse()

        monkeypatch.setattr(requests, "get", mock_get)


    # notice our test uses the custom fixture instead of monkeypatch directly
    def test_get_json(mock_response):
        result = app.get_json("https://fakeurl")
        assert result["mock_key"] == "mock_response"


此外，如果希望mock应用于所有测试，可以将 ``fixture`` 移动到 ``conftest.py`` 中，并使用 ``autouse=True`` 选项。


全局补丁示例：防止远程操作的"requests"
------------------------------------------------------------------

如果您想防止"requests"库在所有测试中执行http请求，您可以：

.. code-block:: python

    # contents of conftest.py
    import pytest


    @pytest.fixture(autouse=True)
    def no_requests(monkeypatch):
        """Remove requests.sessions.Session.request for all tests."""
        monkeypatch.delattr("requests.sessions.Session.request")

每个测试函数都会执行这个autouse的fixture，它将删除 ``request.session.Session`` 方法，所以任何尝试在测试中创建http的请求都将失败。


.. note::

    建议不要对内置函数（如 ``open`` 、``compile`` 等）进行修补，因为这可能会破坏pytest的内部机制。
    如果无法避免，增加 ``--tb=native``, ``--assert=plain`` 和 ``--capture=no`` 可能会有所帮助，但不能保证。

.. note::

    注意，修补 ``stdlib`` 函数和pytest使用的某些三方库可能会破坏pytest。
    因此，建议使用 ``MonkeyPatch.context()`` 将修补限制在您想要测试的代码块：

    .. code-block:: python

        import functools


        def test_partial(monkeypatch):
            with monkeypatch.context() as m:
                m.setattr(functools, "partial", 3)
                assert functools.partial == 3

    详细信息请参阅 `#3290 <https://github.com/pytest-dev/pytest/issues/3290>`_ 。


Monkeypatching环境变量
------------------------------------

如果您正在处理环境变量，为了测试的目的，您经常需要安全地更改这些值或从系统中删除它们。monkeypatch提供了一种使用setenv和delenv方法实现这一点的机制。我们要测试的示例代码:
出于测试的目的，您可能需要从系统中安全地修改或删除环境变量。 ``monkeypatch`` 通过 ``setenv`` 和 ``delenv`` 方法实现该机制。
测试示例代码：

.. code-block:: python

    # contents of our original code file e.g. code.py
    import os


    def get_os_user_lower():
        """Simple retrieval function.
        Returns lowercase USER or raises EnvironmentError."""
        username = os.getenv("USER")

        if username is None:
            raise EnvironmentError("USER environment is not set.")

        return username.lower()

有两种可能性：一、环境变量 ``USER`` 被设定值。二、环境变量 ``USER`` 不存在。
使用 ``monkeypatch`` 可以安全地测试这两种情况，而不影响运行环境。

.. code-block:: python

    # contents of our test file e.g. test_code.py
    import pytest


    def test_upper_to_lower(monkeypatch):
        """Set the USER env var to assert the behavior."""
        monkeypatch.setenv("USER", "TestingUser")
        assert get_os_user_lower() == "testinguser"


    def test_raise_exception(monkeypatch):
        """Remove the USER env var and assert EnvironmentError is raised."""
        monkeypatch.delenv("USER", raising=False)

        with pytest.raises(EnvironmentError):
            _ = get_os_user_lower()

这个行为可以移动到 ``fixture`` 中，并在测试间共享。

.. code-block:: python

    # contents of our test file e.g. test_code.py
    import pytest


    @pytest.fixture
    def mock_env_user(monkeypatch):
        monkeypatch.setenv("USER", "TestingUser")


    @pytest.fixture
    def mock_env_missing(monkeypatch):
        monkeypatch.delenv("USER", raising=False)


    # notice the tests reference the fixtures for mocks
    def test_upper_to_lower(mock_env_user):
        assert get_os_user_lower() == "testinguser"


    def test_raise_exception(mock_env_missing):
        with pytest.raises(EnvironmentError):
            _ = get_os_user_lower()


Monkeypatching字典
---------------------------

使用 :py:meth:`monkeypatch.setitem` 可以在测试期间安全地将字典的值设为特定值。以这个简化的连接字符串为例：

.. code-block:: python

    # contents of app.py to generate a simple connection string
    DEFAULT_CONFIG = {"user": "user1", "database": "db1"}


    def create_connection_string(config=None):
        """Creates a connection string from input or defaults."""
        config = config or DEFAULT_CONFIG
        return f"User Id={config['user']}; Location={config['database']};"

出于测试的目的，可以将字典 ``DEFAULT_CONFIG`` 修补为特定的值。

.. code-block:: python

    # contents of test_app.py
    # app.py with the connection string function (prior code block)
    import app


    def test_connection(monkeypatch):

        # Patch the values of DEFAULT_CONFIG to specific
        # testing values only for this test.
        monkeypatch.setitem(app.DEFAULT_CONFIG, "user", "test_user")
        monkeypatch.setitem(app.DEFAULT_CONFIG, "database", "test_db")

        # expected result based on the mocks
        expected = "User Id=test_user; Location=test_db;"

        # the test uses the monkeypatched dictionary settings
        result = app.create_connection_string()
        assert result == expected

可以使用 ``monkeypatch.delitem()`` 删除值。

.. code-block:: python

    # contents of test_app.py
    import pytest

    # app.py with the connection string function
    import app


    def test_missing_user(monkeypatch):

        # patch the DEFAULT_CONFIG t be missing the 'user' key
        monkeypatch.delitem(app.DEFAULT_CONFIG, "user", raising=False)

        # Key error expected because a config is not passed, and the
        # default is now missing the 'user' entry.
        with pytest.raises(KeyError):
            _ = app.create_connection_string()


Fixture的模块化使您能够灵活地为每个mock单独定义fixture，并在测试中引用它们。

.. code-block:: python

    # contents of test_app.py
    import pytest

    # app.py with the connection string function
    import app

    # all of the mocks are moved into separated fixtures
    @pytest.fixture
    def mock_test_user(monkeypatch):
        """Set the DEFAULT_CONFIG user to test_user."""
        monkeypatch.setitem(app.DEFAULT_CONFIG, "user", "test_user")


    @pytest.fixture
    def mock_test_database(monkeypatch):
        """Set the DEFAULT_CONFIG database to test_db."""
        monkeypatch.setitem(app.DEFAULT_CONFIG, "database", "test_db")


    @pytest.fixture
    def mock_missing_default_user(monkeypatch):
        """Remove the user key from DEFAULT_CONFIG"""
        monkeypatch.delitem(app.DEFAULT_CONFIG, "user", raising=False)


    # tests reference only the fixture mocks that are needed
    def test_connection(mock_test_user, mock_test_database):

        expected = "User Id=test_user; Location=test_db;"

        result = app.create_connection_string()
        assert result == expected


    def test_missing_user(mock_missing_default_user):

        with pytest.raises(KeyError):
            _ = app.create_connection_string()


.. currentmodule:: _pytest.monkeypatch

API参考
-------------

查阅 :class:`MonkeyPatch` 的文档。
