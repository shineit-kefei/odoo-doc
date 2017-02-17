:banner: banners/cmdline.jpg

.. _reference/cmdline:

===============================
命令行界面: odoo-bin
===============================

.. _reference/cmdline/server:

启动server
==================

.. program:: odoo-bin

.. option:: -d <数据库名>, --database <数据库名>

    当安装或更新模块是使用数据库。

.. option:: -i <模块>, --init <模块>

    在运行服务之前用逗号分隔一系列安装的模块
    (要求:option:`-d`).

.. option:: -u <模块>, --update <模块>

    在运行服务之前逗号分隔一系列模块来更新
    (要求 :option:`-d`).

.. option:: --addons-path <目录>

    逗号分隔存储模块的目录。扫描这些目录中的模块

.. option:: --workers <数目>

    如果``计数``不是0（默认），使多处理器和设置指定数量的HTTP角色（子流程处理HTTP和RPC请求）

    .. note:: 多处理器模式仅在基于Unix系统下可用

    一系列选项允许限制和循环角色:

    .. option:: --limit-request <限制数目>

        在回收和重启之前，一个角色要处理的请求数。

        默认是 8196.

    .. option:: --limit-memory-soft <限制内存>

        每个角色允许最大限度的虚拟内存。如果限制被超出，这个角色将被停止并且在当前请求的最后被再回收。

        默认是 640MB.

    .. option:: --limit-memory-hard <限制内存>

        强制限制虚拟内存，并且任意角色超出限制将被立刻停止而不必等待当前请求进程的结束处理。

        默认是 768MB.

    .. option:: --limit-time-cpu <限制秒数>

        防止该角色使用每个请求超过限制CPU的秒数。如果超出限额，角色将被停止。


        默认是 60.

    .. option:: --limit-time-real <限制时间>

        防止角色花费超过限制秒数的处理请求。如果超出限额，角色将被停止。

        不同于 :option:`--limit-time-cpu` ，这是一个"硬性时间"限制包括例如 SQL查询。

        默认是 120.

.. option:: --max-cron-threads <count>

    一系列角色专用于计划任务。默认是2。这些角色在多线程模式下是线程，在多进程模式下是进程。

    针对多进程模式，这被加入到HTTP角色进程中。

.. option:: -c <配置文件>, --config <配置文件>

    通过一个转换的配置文件

.. option:: -s, --save

    保存服务器配置到当前的配置文件中(默认是 :file:`{$HOME}/.odoorc` ，并且使用
    :option: `-c` 被覆盖)

.. option:: --proxy-mode

    enables the use of ``X-Forwarded-*`` headers through `Werkzeug's proxy
    support`_.

    .. warning:: proxy mode *must not* be enabled outside of a reverse proxy
                 scenario

.. option:: --test-enable

    安转模块后运行测试

.. option:: --dev <feature,feature,...,feature>

    * ``all``: 以下的所有功能特性被激活

    * ``xml``: 从xml文件直接读取qweb模板，而不是从数据库中。一旦一个模板在数据库中被改动，它将不会从
      xml文件中被读取到直至下次更新/初始化。

    * ``reload``: 当python文件被更新重启服务(可能无法检测取决于使用的文本编辑器)

    * ``qweb``: 当一个节点包含 ``t-debug='debugger'`` 时相关的qweb模板被中断。

    * ``(i)p(u)db``:当一个意外的错误出现而日志返回这个错误之前启动一个python的调试器。

.. _reference/cmdline/server/database:

database
--------

.. option:: -r <数据库用户>, --db_user <数据库用户>

    用户连接PostgreSQL的数据库用户。

.. option:: -w <数据库密码>, --db_password <数据库密码>

    如果使用 `密码验证` ，提供数据库密码。

.. option:: --db_host <主机地址>

    数据库服务器地址

    * ``localhost`` on Windows
    * UNIX socket otherwise

.. option:: --db_port <端口号>

    数据库监听端口号，默认是 5432

.. option:: --db-filter <筛选器>

    隐藏不匹配<筛选器> 的数据库。过滤器是一种 `正则表达式` ，补充:

    - ``%h`` 被请求的整个主机名替换
    - ``%d`` 被替换为请求中除www以外的子域名(因此域odoo.com和www.odoo.com都将匹配odoo数据库)

.. option:: --db-template <模板>

    当从数据库管理界面创建一个新的数据库，使用指定的 `template database` 。 
    默认是 ``template1`` 。

built-in HTTP
-------------

.. option:: --no-xmlrpc

    do not start the HTTP or long-polling workers (may still start cron
    workers)

    .. warning:: has no effect if :option:`--test-enable` is set, as tests
                 require an accessible HTTP server

.. option:: --xmlrpc-interface <interface>

    TCP/IP address on which the HTTP server listens, defaults to ``0.0.0.0``
    (all addresses)

.. option:: --xmlrpc-port <port>

    Port on which the HTTP server listens, defaults to 8069.

.. option:: --longpolling-port <port>

    TCP port for long-polling connections in multiprocessing or gevent mode,
    defaults to 8072. Not used in default (threaded) mode.

logging
-------

By default, Odoo displays all logging of level_ ``info`` except for workflow
logging (``warning`` only), and log output is sent to ``stdout``. Various
options are available to redirect logging to other destinations and to
customize the amount of logging output

.. option:: --logfile <file>

    sends logging output to the specified file instead of stdout. On Unix, the
    file `can be managed by external log rotation programs
    <https://docs.python.org/2/library/logging.handlers.html#watchedfilehandler>`_
    and will automatically be reopened when replaced

.. option:: --logrotate

    enables `log rotation <https://docs.python.org/2/library/logging.handlers.html#timedrotatingfilehandler>`_
    daily, keeping 30 backups. Log rotation frequency and number of backups is
    not configurable.

.. option:: --syslog

    logs to the system's event logger: `syslog on unices <https://docs.python.org/2/library/logging.handlers.html#sysloghandler>`_
    and `the Event Log on Windows <https://docs.python.org/2/library/logging.handlers.html#nteventloghandler>`_.

    Neither is configurable

.. option:: --log-db <dbname>

    logs to the ``ir.logging`` model (``ir_logging`` table) of the specified
    database. The database can be the name of a database in the "current"
    PostgreSQL, or `a PostgreSQL URI`_ for e.g. log aggregation

.. option:: --log-handler <handler-spec>

    :samp:`{LOGGER}:{LEVEL}`, enables ``LOGGER`` at the provided ``LEVEL``
    e.g. ``odoo.models:DEBUG`` will enable all logging messages at or above
    ``DEBUG`` level in the models.

    * The colon ``:`` is mandatory
    * The logger can be omitted to configure the root (default) handler
    * If the level is omitted, the logger is set to ``INFO``

    The option can be repeated to configure multiple loggers e.g.

    .. code-block:: console

        $ odoo-bin --log-handler :DEBUG --log-handler werkzeug:CRITICAL --log-handler odoo.fields:WARNING

.. option:: --log-request

    enable DEBUG logging for RPC requests, equivalent to
    ``--log-handler=odoo.http.rpc.request:DEBUG``

.. option:: --log-response

    enable DEBUG logging for RPC responses, equivalent to
    ``--log-handler=odoo.http.rpc.response:DEBUG``

.. option:: --log-web

    enables DEBUG logging of HTTP requests and responses, equivalent to
    ``--log-handler=odoo.http:DEBUG``

.. option:: --log-sql

    enables DEBUG logging of SQL querying, equivalent to
    ``--log-handler=odoo.sql_db:DEBUG``

.. option:: --log-level <level>

    Shortcut to more easily set predefined levels on specific loggers. "real"
    levels (``critical``, ``error``, ``warn``, ``debug``) are set on the
    ``odoo`` and ``werkzeug`` loggers (except for ``debug`` which is only
    set on ``odoo``).

    Odoo also provides debugging pseudo-levels which apply to different sets
    of loggers:

    ``debug_sql``
        sets the SQL logger to ``debug``

        equivalent to ``--log-sql``
    ``debug_rpc``
        sets the ``odoo`` and HTTP request loggers to ``debug``

        equivalent to ``--log-level debug --log-request``
    ``debug_rpc_answer``
        sets the ``odoo`` and HTTP request and response loggers to
        ``debug``

        equivalent to ``--log-level debug --log-request --log-response``

    .. note::

        In case of conflict between :option:`--log-level` and
        :option:`--log-handler`, the latter is used


.. _reference/cmdline/scaffold:

Scaffolding
===========

.. program:: odoo-bin scaffold

Scaffolding is the automated creation of a skeleton structure to simplify
bootstrapping (of new modules, in the case of Odoo). While not necessary it
avoids the tedium of setting up basic structures and looking up what all
starting requirements are.

Scaffolding is available via the :command:`odoo-bin scaffold` subcommand.

.. option:: -t <template>

    a template directory, files are passed through jinja2_ then copied to
    the ``destination`` directory

.. option:: name

    the name of the module to create, may munged in various manners to
    generate programmatic names (e.g. module directory name, model names, …)

.. option:: destination

    directory in which to create the new module, defaults to the current
    directory

.. _reference/cmdline/config:

Configuration file
==================

Most of the command-line options can also be specified via a configuration
file. Most of the time, they use similar names with the prefix ``-`` removed
and other ``-`` are replaced by ``_`` e.g. :option:`--db-template` becomes
``db_template``.

Some conversions don't match the pattern:

* :option:`--db-filter` becomes ``dbfilter``
* :option:`--no-xmlrpc` corresponds to the ``xmlrpc`` boolean
* logging presets (all options starting with ``--log-`` except for
  :option:`--log-handler` and :option:`--log-db`) just add content to
  ``log_handler``, use that directly in the configuration file
* :option:`--smtp` is stored as ``smtp_server``
* :option:`--database` is stored as ``db_name``
* :option:`--debug` is stored as ``debug_mode`` (a boolean)
* :option:`--i18n-import` and :option:`--i18n-export` aren't available at all
  from configuration files

The default configuration file is :file:`{$HOME}/.odoorc` which
can be overridden using :option:`--config <odoo-bin -c>`. Specifying
:option:`--save <odoo-bin -s>` will save the current configuration state back
to that file.

.. _jinja2: http://jinja.pocoo.org
.. _regular expression: https://docs.python.org/2/library/re.html
.. _password authentication:
    http://www.postgresql.org/docs/9.3/static/auth-methods.html#AUTH-PASSWORD
.. _template database:
    http://www.postgresql.org/docs/9.3/static/manage-ag-templatedbs.html
.. _level:
    https://docs.python.org/2/library/logging.html#logging.Logger.setLevel
.. _a PostgreSQL URI:
    http://www.postgresql.org/docs/9.2/static/libpq-connect.html#AEN38208
.. _Werkzeug's proxy support:
    http://werkzeug.pocoo.org/docs/contrib/fixers/#werkzeug.contrib.fixers.ProxyFix
.. _pyinotify: https://github.com/seb-m/pyinotify/wiki
