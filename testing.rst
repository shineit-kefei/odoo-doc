:banner: banners/testing_modules.jpg

.. _reference/testing:


===============
测试模块
===============


Odoo 提供使用unittest之策测试模块。

为了写tests，在你的模块里简单定义一个 ``tests`` 子目录，它将被自动检测为测试模块。
测试模块应该以 ``test_`` 开始并且从 ``tests/__init__.py`` 导入，
例如

.. code-block:: text

    your_module
    |-- ...
    `-- tests
        |-- __init__.py
        |-- test_bar.py
        `-- test_foo.py

并且 ``__init__.py`` 包括::

    from . import test_foo, test_bar

.. warning::

    未从 ``tests/__init__.py`` 导入的测试模块将不会运行

.. versionchanged:: 8.0

    以前，测试者将会运行加入 ``fast_suite`` 和 ``checks`` 到 ``tests/__init__.py``
    的模块。在 8.0 会运行所有被导入的模块

测试者将简单运行任意测试案例，在官方`unittest documentation`_ 作为描述， 但是Odoo提供了一系列
实用工具并且辅助物体来关联测试Odoo内容（modules, mainly):

.. autoclass:: odoo.tests.common.TransactionCase
    :members: browse_ref, ref

.. autoclass:: odoo.tests.common.SingleTransactionCase
    :members: browse_ref, ref

默认情况，测试运行一次后相应的模块应该已经被安装，并且在模块安转后不运行:

.. autofunction:: odoo.tests.common.at_install

.. autofunction:: odoo.tests.common.post_install


:class:`~odoo.tests.common.TransactionCase` 是被最常使用的并且在每个方法中测试模型的一个属性::

    class TestModelA(common.TransactionCase):
        def test_some_action(self):
            record = self.env['model.a'].create({'field': 'value'})
            record.some_action()
            self.assertEqual(
                record.field,
                expected_field_value)

        # other tests...

运行测试
-------------

重启odoo服务时，如果 :option:`--test-enable <odoo-bin --test-enable>` 被可用，当安装或更新模块时
测试将自动运行

作为odoo 8，不支持安装/更新周期以外来运行测试

.. _unittest documentation: https://docs.python.org/2/library/unittest.html
