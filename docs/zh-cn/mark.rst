.. _mark:

使用属性标记测试函数
======================================

通过使用 ``pytest.mark`` ，您可以在测试函数上轻松地设置元数据。有一些内置标记，例如：

* :ref:`skip <skip>` - 总是跳过测试函数
* :ref:`skipif <skipif>` - 如果满足某个条件，则跳过测试函数
* :ref:`xfail <xfail>` - 如果满足某个条件，则产生"预期失败"结果
* :ref:`parametrize <parametrizemark>` - 对同一测试函数执行多次调用

我们可以很容易创建自定义标记，或将标记应用于整个测试类/模块。这些标记可以由插件使用，
通常也用于命令行中 ``-m`` 选项来 :ref:`选择测试 <mark run>` 。

请参阅 :ref:`mark examples` 查看标记的示例和文档。

.. note::

    标记只能应用于测试，对 :ref:`fixtures <fixtures>` 不起作用。


注册标记
-----------------

您可以在 ``pytest.ini`` 文件中注册自定义标记，如下所示：

.. code-block:: ini

    [pytest]
    markers =
        slow: marks tests as slow (deselect with '-m "not slow"')
        serial

注意， ``:`` 之后的所有内容都是可选说明。

或者，您可以在 :ref:`pytest_configure <initialization-hooks>` 钩子中程序化注册新标记：

.. code-block:: python

    def pytest_configure(config):
        config.addinivalue_line(
            "markers", "env(name): mark test to run only on named environment"
        )


注册标记出现在pytest的帮助文本中，不会产生警告（请参阅下一节）。建议第三方插件始终 :ref:`注册其标记 <registering-markers>`

.. _unknown-marks:

对未知标记提出错误
-------------------------------

使用 ``@pytest.mark.name_of_the_mark`` 装饰器应用未注册标记总是会发出警告，以避免由于名称键入错误而导致的意外操作。
如前一节所述，可以通过在 ``pytest.ini`` 文件中注册自定义的标记，或使用 ``pytest_configure`` 钩子来禁用警告。


当使用 ``--strict-markers`` 命令行标志时， 任何使用 ``@pytest.mark.name_of_the_mark`` 装饰器的未知标记都将触发一个错误。
您可以通过在 ``addopts`` 里面添加 ``--strict-marker`` 在项目中验证：

.. code-block:: ini

    [pytest]
    addopts = --strict-markers
    markers =
        slow: marks tests as slow (deselect with '-m "not slow"')
        serial
