:orphan:

.. _features:

pytest: 助你更好编程
=======================================


``pytest`` 框架可以轻松编写小型测试，同时支持应用程序和软件库的复杂功能性测试。

一个简单的例子：

.. code-block:: python

    # content of test_sample.py
    def inc(x):
        return x + 1


    def test_answer():
        assert inc(3) == 5


执行：

.. code-block:: pytest

    $ pytest
    =========================== test session starts ============================
    platform linux -- Python 3.x.y, pytest-5.x.y, py-1.x.y, pluggy-0.x.y
    cachedir: $PYTHON_PREFIX/.pytest_cache
    rootdir: $REGENDOC_TMPDIR
    collected 1 item

    test_sample.py F                                                     [100%]

    ================================= FAILURES =================================
    _______________________________ test_answer ________________________________

        def test_answer():
    >       assert inc(3) == 5
    E       assert 4 == 5
    E        +  where 4 = inc(3)

    test_sample.py:6: AssertionError
    ========================= 1 failed in 0.12 seconds =========================

由于 ``pytest`` 有详细的断言说明，所以只简单使用 ``assert`` 语句。有关更多示例，请参阅 :ref:`入门 <getstarted>`


特性
--------

- 失败的 :ref:`断言语句<assert>` 详细信息（无需记住 ``self.assert*`` 名称）;

- :ref:`自动发现 <test discovery>` 测试模块和功能;

- :ref:`模块化fixtures <fixture>` 用于管理小型或参数化长生命周期的测试资源;

- 可以运行 :ref:`unittest <unittest>` （包括试用期）,并且可以开箱即用测试套件 :ref:`nose <noseintegration>`;

- Python 3.5+ and PyPy 3;

- 丰富的插件架构，拥有超过315+ `外部插件 <http://plugincompat.herokuapp.com>`_ 的蓬勃发展的社区;


文档
-------------

请参阅 :ref:`内容 <toc>` 中的完整文档，包括安装，教程和PDF文档。


错误/请求
-------------

请使用 `GitHub问题追踪 <https://github.com/pytest-dev/pytest/issues>`_ 来提交错误或请求特性。


变更日志
---------

请参阅 :ref:`变更日志 <changelog>` 页面，有关于每个版本的修复和增强功能。


许可证
-------

版权所有Holger Krekel等，2004-2017。

根据 `MIT`_ 许可条款发布，pytest是免费的开源软件。

.. _`MIT`: https://github.com/pytest-dev/pytest/blob/master/LICENSE
