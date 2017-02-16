:banner: banners/orm_api.jpg

.. _reference/orm:

=======
ORM API
=======

记录集
==========

.. versionadded:: 8.0

    本文档的新API应该作为odoo8.0以后主要的API。它还提供了有关移植或桥接7.0和
    更早版本的“旧API”的信息，但该API没有明确地文档。看到旧的文档。

通过同一模块排序的记录集将模块和记录的相互作用表现出来。

.. warning:: 相反，顾名思义，目前可能记录集里包含重复。这可能会在未来改变。

在一个记录集上执行模块上定义的方法，并且``self``是一个记录集::

    class AModel(models.Model):
        _name = 'a.model'
        def a_method(self):
            # self can be anywhere between 0 records and all records in the
            # database
            self.do_operation()

迭代一个记录集将产生新的*一个单独记录*的集合("singletons"),就像一个Python字符串迭代
生成一个单一字符串::

        def do_operation(self):
            print self # => a.model(1, 2, 3, 4, 5)
            for record in self:
                print record # => a.model(1), then a.model(2), then a.model(3), ...

字段访问
------------

记录集提供一个“激活记录”界面: 模块字段可以通过这个记录直接被读和写，但是仅限单一记录集(signle-record)。
设置一个字段的值将触发一次数据库更新::

    >>> record.name
    Example Name
    >>> record.company_id.name
    Company Name
    >>> record.name = "Bob"

尝试在多个记录中读取或写入字段会激发一个错误。

访问一个关联字段(:class:`~odoo.fields.Many2one`,
:class:`~odoo.fields.One2many`, :class:`~odoo.fields.Many2many`) *总是* 返回一个记录集，
如果该字段没有值将返回空的记录集。

.. danger::

    每次赋值给一个字段将触发一次数据库更新, 当同一时间设置多个字段或给多个记录
    设置字段(赋相同的值), 使用:meth:`~odoo.models.Model.write`::

        # 3 * len(records) database updates
        for record in records:
            record.a = 1
            record.b = 2
            record.c = 3

        # len(records) database updates
        for record in records:
            record.write({'a': 1, 'b': 2, 'c': 3})

        # 1 database update
        records.write({'a': 1, 'b': 2, 'c': 3})


记录缓存和预取
----------------

Odoo为记录中的字段维持一个缓存空间，因此不是每次访问字段都触发数据库请求，
不然那将对性能的影响是可怕的。这个下面例子只对第一个语句进行数据库查询::

    record.name             # first access reads value from database
    record.name             # second access gets value from cache

为了避免每次读一个记录上的一个字段，Odoo *预取* 记录和字段借鉴一个启发式算法以获得更好的性能。
一旦一个字段必须在给定的记录上被读取，这个ORM实际在一个大的记录集上读取该字段，并且以便下一个
用户读取，将该字段返回的值到存储到缓存中，所有基础存储的字段(boolean, integer, float, char,
text, date, datetime, selection, many2one)被同时获取到；它们对应于模型表的列，并在同一
查询中高效地获取。

考虑下面这个例子, ``partners`` 是一个有1000条记录的记录集。如果没有预取,该循环将对数据库
进行2000次查询,如果有预取,仅仅需要查询一次::

    for partner in partners:
        print partner.name          # first pass prefetches 'name' and 'lang'
                                    # (and other fields) on all 'partners'
        print partner.lang

预取也适用于 *二次记录* ：当关系字段被读取时，它们的值（这是记录）为将来的预取订阅。其中访问二次
记录中的某个字段只需从相同的模型预取所有次要记录。这使得下面的示例只生成两次查询，
一个用于合作伙伴，另一个用于国家::

    countries = set()
    for partner in partners:
        country = partner.country_id        # first pass prefetches all partners
        countries.add(country.name)         # first pass prefetches all countries


集合操作
-------------

记录集是不变的，但是相同模块的集合可以通过使用一系列的操作符来结合，返回新的记录集，
集合操作 *不* 保留顺序

.. addition preserves order but can introduce duplicates

.. 增加保存订单但是可以重复


* ``record in set`` 是否返回 ``record`` (第一项必须是一个元素的记录集) 
  在 ``set`` 中 。 ``record not in set`` 是相反的操作
* ``set1 <= set2`` 和 ``set1 < set2`` 返回是否 ``set1`` 是 ``set2`` 的子集
* ``set1 >= set2`` 和 ``set1 > set2`` 返回是否 ``set1`` 是 ``set2`` 的超级
* ``set1 | set2`` 返回两个集合的合集,一个新的记录集包括任一个集合中的所有记录
* ``set1 & set2`` 返回两个集合的并集,一个新的记录集仅包括两个集合中都有的记录
* ``set1 - set2`` 返回一个记录在``set1``而不 ``set2`` 中的新的记录集


其他记录集操作方法
------------------


记录集是可迭代的因此通常的Python 工具是可用于转化的(:func:`python:map`, :func:`python:sorted`,
:func:`~python:itertools.ifilter`, ...) 然而这些返回的要么是:class:`python:list`
要么是:term:`python:iterator`,去除了在结果之上调用的方法的能力，或者去除了使用集合的操作。


记录集因此提供这些操作返回记录集本身:


:meth:`~odoo.models.Model.filtered`
    返回一个只包含满足提供判定函数的记录集。判定也可以是由真或假字段筛选的字符串::

        # only keep records whose company is the current user's
        records.filtered(lambda r: r.company_id == user.company_id)

        # only keep records whose partner is a company
        records.filtered("partner_id.is_company")

:meth:`~odoo.models.Model.sorted`
    返回一个通过关键字函数排序的记录集。如果未提供关键字，使用模块默认的排序::

        # sort records by name
        records.sorted(key=lambda r: r.name)

:meth:`~odoo.models.Model.mapped`
    将提供的函数应用于记录集中的每一条记录，如果结果是记录集将返回一个记录集::

        # returns a list of summing two fields for each record in the set
        records.mapped(lambda r: r.field1 + r.field2)

    提供的函数可以使字符串来获取字段的值::

        # returns a list of names
        records.mapped('name')

        # returns a recordset of partners
        record.mapped('partner_id')

        # returns the union of all partner banks, with duplicates removed
        record.mapped('partner_id.bank_ids')


环境
=====

:class:`~odoo.api.Environment` 通过ORM存储各种环境中的数据：数据库游标(数据库查询) 、
当前用户 (用来权限检查)、当前环境(存储任意的元数据)。环境可以被存储在缓存中。

所有的记录集都有一个不可变的环境，它可以通过 :attr:`~odoo.models.Model.env` 访问,通过
(:attr:`~odoo.api.Environment.user`), 游标
(:attr:`~odoo.api.Environment.cr`) or 上下文
(:attr:`~odoo.api.Environment.context`)给当前用户访问::

    >>> records.env
    <Environment object ...>
    >>> records.env.user
    res.user(3)
    >>> records.env.cr
    <Cursor object ...)

当从其他记录集创建了一个记录集，这个环境是可以被继承的。环境可以被用于从其他模块获取一个空的记录集，
并且查询这个模块::

    >>> self.env['res.partner']
    res.partner
    >>> self.env['res.partner'].search([['is_company', '=', True], ['customer', '=', True]])
    res.partner(7, 18, 12, 14, 17, 19, 8, 31, 26, 16, 13, 20, 30, 22, 29, 15, 23, 28, 74)

转换环境
---------

来自一个记录集的环境可以被定制。使用转换环境返回一个新版本的记录集。


:meth:`~odoo.models.Model.sudo`
    根据提供的用户来创建一个新的环境，如果没有提供则使用管理员（绕过权限/规则的安全上下文），
    返回一个调用使用新的环境的记录集::

        # create partner object as administrator
        env['res.partner'].sudo().create({'name': "A Partner"})

        # list partners visible by the "public" user
        public = env.ref('base.public_user')
        env['res.partner'].sudo(public).search([])


:meth:`~odoo.models.Model.with_context`
    #. 可以携带一个位置参数，它将取代目前的环境的上下文
    #. 可以通过关键字携带任意数量的参数，这些参数将被增加到当前环境上下文中或步骤1中的上下文设置中::

        # look for partner, or create one with specified timezone if none is
        # found
        env['res.partner'].with_context(tz=a_tz).find_or_create(email_address)


:meth:`~odoo.models.Model.with_env`
    彻底替换现存的环境


共用的ORM方法
================

.. 也许这些说明/例子应该在API文档中？


:meth:`~odoo.models.Model.search`
    提供一个:ref:`search domain <reference/orm/domains>`,返回一个匹配的记录集。
    也可以返回匹配的记录集的子集,通过 ``offset`` 和 ``limit`` 参数,同时通过 ``order`` 参数排序::

        >>> # searches the current model
        >>> self.search([('is_company', '=', True), ('customer', '=', True)])
        res.partner(7, 18, 12, 14, 17, 19, 8, 31, 26, 16, 13, 20, 30, 22, 29, 15, 23, 28, 74)
        >>> self.search([('is_company', '=', True)], limit=1).name
        'Agrolait'

   .. tip:: 只检查是否有任何匹配的记录,或计数的数目,使用
             :meth:`~odoo.models.Model.search_count`


:meth:`~odoo.models.Model.create`
    提供一系列数量字段的值，返回包含该记录的记录集::

        >>> self.create({'name': "New Name"})
        res.partner(78)


:meth:`~odoo.models.Model.write`
    提供一系列字段值，将他们写入到记录集中的所有记录中。不返回任何东西::

        self.write({'name': "Newer Name"})


:meth:`~odoo.models.Model.browse`    
    提供数据库id或者ids的集合，返回一个记录集，当记录的id从odoo之外被获取是有用的
    (例如 往返通过外部的系统)或:ref:`在旧的API中调用方法
    <reference/orm/oldapi>`::

        >>> self.browse([7, 18, 12])
        res.partner(7, 18, 12)

    
:meth:`~odoo.models.Model.exists`
    返回一个仅存在于数据库中记录的新的记录集。可以用来检查是否该记录依旧存在::

        if not record.exists():
            raise Exception("The record has been deleted")

    或者调用一个方法后应该移除一些记录::

        records.may_remove_some()
        # only keep records which were not deleted
        records = records.exists()

    
:meth:`~odoo.api.Environment.ref`
    环境的方法返回匹配到的:term:`external id` 的记录::
    
        >>> env.ref('base.group_public')
        res.groups(2)
    
:meth:`~odoo.models.Model.ensure_one`
    检查该记录集是一个signleton(仅包含一个单一记录)，否则提示一个错误::

        records.ensure_one()
        # is equivalent to but clearer than:
        assert len(records) == 1, "Expected singleton"


创建模块
===============


模块字段作为属性被定义在模块上::

    from odoo import models, fields
    class AModel(models.Model):
        _name = 'a.model.name'

        field1 = fields.Char()


.. warning:: 意思你不能用同一个名字定义一个字段和方法，它们会冲突


默认字段标签（用户可见的名字）是该字段首字母大写的名字，它可以被``string``属性复写::

        field2 = fields.Integer(string="an other field")


针对不同类型字段和参数，看:ref:`参考该字段
<reference/orm/fields>`


默认值作为参数被定义在字段上，要么是一个值::

    a_field = fields.Char(default="a value")

要么是一个被调用来计算默认值的函数，该函数应该返回那个值::

    def compute_default_value(self):
        return self.get_value()
    a_field = fields.Char(default=compute_default_value)

计算字段
---------------


字段可以被计算(而不是直接从数据库读出来)使用``compute``参数。**它必须将计算的值赋值给该字段**。
如果它使用其他*字段*的值，它应该指明这些字段使用:func:`~odoo.api.depends`::

    from odoo import api
    total = fields.Float(compute='_compute_total')

    @api.depends('value', 'tax')
    def _compute_total(self):
        for record in self:
            record.total = record.value + record.value * record.tax

* 当使用子字段时可以使用点操作::

    @api.depends('line_ids.value')
    def _compute_total(self):
        for record in self:
            record.total = sum(line.value for line in record.line_ids)

* 计算字段默认不被存储，当请求时它们被计算和返回。设置``store=True``将被存储在数据库中
  并可以自动被搜索

* 通过设置``search``参数来在计算字段上搜索。该值是一个方法的名字返回一个
  :ref:`reference/orm/domains`::

    upper_name = field.Char(compute='_compute_upper', search='_search_upper')

    def _search_upper(self, operator, value):
        if operator == 'like':
            operator = 'ilike'
        return [('name', operator, value)]

* 允许*设置*值到计算字段，使用``inverse``参数。它是一个函数的名称反转计算并且设置关联字段::

    document = fields.Char(compute='_get_document', inverse='_set_document')

    def _get_document(self):
        for record in self:
            with open(record.get_document_path) as f:
                record.document = f.read()
    def _set_document(self):
        for record in self:
            if not record.document: continue
            with open(record.get_document_path()) as f:
                f.write(record.document)

* 多个字段可以在同一时间通过同一方法被计算出来, 仅仅使用相同的方法在所有的字段上
  并且设置它们::

    discount_value = fields.Float(compute='_apply_discount')
    total = fields.Float(compute='_apply_discount')

    @depends('value', 'discount')
    def _apply_discount(self):
        for record in self:
            # compute actual discount from discount percentage
            discount = record.value * record.discount
            record.discount_value = discount
            record.total = record.value - discount

关联字段
'''''''

一个特殊的计算字段是 *related* (代理)字段,该字段在当前记录提供一个子字段的值.
他们通过 ``related`` 参数被指定并且像常规的计算字段可以被存储::

    nickname = fields.Char(related='user_id.partner_id.name', store=True)



onchange: 运行中更新界面
--------------------------

当一个用户在表单视图改变一个字段值（但是还没保存），它可以基于这个值被用于自动更新其他字段
例如，当税收改变或者添加一个新的发票行将更新最终总数。

* 计算字段会自动检查并重新计算，他们不需要onchange
* 对于非计算字段，这个onchange()装饰器被用于提供新字段::

    @api.onchange('field1', 'field2') # if these fields are changed, call method
    def check_change(self):
        if self.field1 < self.field2:
            self.field3 = True

  在这个方法被发送至客户端程序并且对用户可视时，这个改变被展现出来

* 计算字段和新的onchanges API通过客户端被自动调用无需添加他们在视图中
* 通过添加on_change="0” 在view中来抑制一个指定字段触发onchange方法::

    <field name="name" on_change="0"/>

  当该字段被用户编辑时，将不会触发任何界面更新，即使这有个函数字段或明确的onchange依赖于该字段

.. note::

    ``onchange`` 方法运行在虚拟记录赋值这些记录是不写入数据库,只是用来知道这值返回给客户


底层SQL
----------------

:attr:`~odoo.api.Environment.cr` 环境属性是针对当前事务的游标和允许直接执行SQL,要么是难以使用
ORM来进行表达, 要么是性能的原因::

    self.env.cr.execute("some_sql", param1, param2, param3)


由于模块使用相同的游标并且 :class:`~odoo.api.Environment` 保存不同的缓存，当在原生的SQL中
 *变更* 数据库时这些缓存必须失效，或者进一步利用模块来变成不相干。当在SQL中使用 ``CREATE``, ``UPDATE``
或者 ``DELETE`` 时必须清除缓存, 在 ``SELECT`` 中则不需要(这只是简单的读取数据库)。


清除缓存可以使用 :class:`~odoo.api.Environment` 对象中的
:meth:`~odoo.api.Environment.invalidate_all` 方法。


.. _reference/orm/oldapi:

新旧API的兼容性
===============

Odoo目前正从旧的API中过渡，手动通旧API建立桥梁是必须的:

* RPC 层(XML-RPC和JSON-RPC)是用旧的API来表示的，单纯使用新的API表达式方法是不可用RPC的
* 从旧系列的代码重写被调用的方法仍然需要使用旧API的风格

新旧API之间最大的不同:

* :class:`~odoo.api.Environment`(游标，用户id和上下文)的值直接通过方法传递
* 记录数据(:attr:`~odoo.models.Model.ids`)通过明确的方法传递，而且根本没有通过
* 方法趋向使用列表而不是记录集


默认情况下，方法假定使用新API风格而不是调用旧的API。


.. tip:: 从新到旧API的调用是桥接的
    :class: aphorism


    当使用新的API风格，使用旧的API定义的方法被调用时在运行中将自动转换，不需要做任何指定::

        >>> # method in the old API style
        >>> def old_method(self, cr, uid, ids, context=None):
        ...    print ids

        >>> # method in the new API style
        >>> def new_method(self):
        ...     # system automatically infers how to call the old-style
        ...     # method from the new-style method
        ...     self.old_method()

        >>> env[model].browse([1, 2, 3, 4]).new_method()
        [1, 2, 3, 4]


两个装饰器可以使新风格的方法运用到旧的API中:

:func:`~odoo.api.model`
    该方法没有使用ids被展现出来，它的记录集是空的，对应旧的API的 ``cr, uid, *arguments, context``::

        @api.model
        def some_method(self, a_value):
            pass
        # can be called as
        old_style_model.some_method(cr, uid, a_value, context=context)

:func:`~odoo.api.multi`
    该方法携带一系列ids被展现出来，对应旧的API的 ``cr, uid, ids, *arguments, context``::

        @api.multi
        def some_method(self, a_value):
            pass
        # can be called as
        old_style_model.some_method(cr, uid, [id1, id2], a_value, context=context)

由于新风格的APIs趋向返回记录集而旧的APIs趋向返回一系列id的列表，因此这也有管理这个的装饰器:

:func:`~odoo.api.returns`
    该函数被假定返回一个记录集，第一个参数应该被命名为记录集的模块或者 ``self`` (针对当前模块)

    如果该方法在新的API风格中被调用时没有作用的，但是当调用旧的API风格时将记录集转换为一系列id的列表::

        >>> @api.multi
        ... @api.returns('self')
        ... def some_method(self):
        ...     return self
        >>> new_style_model = env['a.model'].browse(1, 2, 3)
        >>> new_style_model.some_method()
        a.model(1, 2, 3)
        >>> old_style_model = pool['a.model']
        >>> old_style_model.some_method(cr, uid, [1, 2, 3], context=context)
        [1, 2, 3]

.. _reference/orm/model:


模块参考
==========

.. - can't get autoattribute to import docstrings, so use regular attribute
   - no autoclassmethod

.. currentmodule:: odoo.models

.. autoclass:: odoo.models.Model

    .. rubric:: Structural attributes

    .. attribute:: _name

        business object name, in dot-notation (in module namespace)

    .. attribute:: _rec_name

        Alternative field to use as name, used by osv’s name_get()
        (default: ``'name'``)

    .. attribute:: _inherit

        * If :attr:`._name` is set, names of parent models to inherit from.
          Can be a ``str`` if inheriting from a single parent
        * If :attr:`._name` is unset, name of a single model to extend
          in-place

        See :ref:`reference/orm/inheritance`.

    .. attribute:: _order

        Ordering field when searching without an ordering specified (default:
        ``'id'``)

        :type: str

    .. attribute:: _auto

        Whether a database table should be created (default: ``True``)

        If set to ``False``, override :meth:`.init` to create the database
        table

    .. attribute:: _table

        Name of the table backing the model created when
        :attr:`~odoo.models.Model._auto`, automatically generated by
        default.

    .. attribute:: _inherits

        dictionary mapping the _name of the parent business objects to the
        names of the corresponding foreign key fields to use::

            _inherits = {
                'a.model': 'a_field_id',
                'b.model': 'b_field_id'
            }

        implements composition-based inheritance: the new model exposes all
        the fields of the :attr:`~odoo.models.Model._inherits`-ed model but
        stores none of them: the values themselves remain stored on the linked
        record.

        .. warning::

            if the same field is defined on multiple
            :attr:`~odoo.models.Model._inherits`-ed

    .. attribute:: _constraints

        list of ``(constraint_function, message, fields)`` defining Python
        constraints. The fields list is indicative

        .. deprecated:: 8.0

            use :func:`~odoo.api.constrains`

    .. attribute:: _sql_constraints

        list of ``(name, sql_definition, message)`` triples defining SQL
        constraints to execute when generating the backing table

    .. attribute:: _parent_store

        Alongside :attr:`~.parent_left` and :attr:`~.parent_right`, sets up a
        `nested set <http://en.wikipedia.org/wiki/Nested_set_model>`_  to
        enable fast hierarchical queries on the records of the current model
        (default: ``False``)

        :type: bool

    .. rubric:: CRUD

    .. automethod:: create
    .. automethod:: browse
    .. automethod:: unlink
    .. automethod:: write

    .. automethod:: read
    .. automethod:: read_group

    .. rubric:: Searching

    .. automethod:: search
    .. automethod:: search_count
    .. automethod:: name_search

    .. rubric:: Recordset operations

    .. autoattribute:: ids
    .. automethod:: ensure_one
    .. automethod:: exists
    .. automethod:: filtered
    .. automethod:: sorted
    .. automethod:: mapped

    .. rubric:: Environment swapping

    .. automethod:: sudo
    .. automethod:: with_context
    .. automethod:: with_env

    .. rubric:: Fields and views querying

    .. automethod:: fields_get
    .. automethod:: fields_view_get

    .. rubric:: ???

    .. automethod:: default_get
    .. automethod:: copy
    .. automethod:: name_get
    .. automethod:: name_create

    .. _reference/orm/model/automatic:

    .. rubric:: Automatic fields

    .. attribute:: id

        Identifier :class:`field <odoo.fields.Field>`

    .. attribute:: _log_access

        Whether log access fields (``create_date``, ``write_uid``, ...) should
        be generated (default: ``True``)

    .. attribute:: create_date

        Date at which the record was created

        :type: :class:`~odoo.field.Datetime`

    .. attribute:: create_uid

        Relational field to the user who created the record

        :type: ``res.users``

    .. attribute:: write_date

        Date at which the record was last modified

        :type: :class:`~odoo.field.Datetime`

    .. attribute:: write_uid

        Relational field to the last user who modified the record

        :type: ``res.users``

    .. rubric:: Reserved field names

    A few field names are reserved for pre-defined behaviors beyond that of
    automated fields. They should be defined on a model when the related
    behavior is desired:

    .. attribute:: name

        default value for :attr:`~._rec_name`, used to
        display records in context where a representative "naming" is
        necessary.

        :type: :class:`~odoo.fields.Char`

    .. attribute:: active

        toggles the global visibility of the record, if ``active`` is set to
        ``False`` the record is invisible in most searches and listing

        :type: :class:`~odoo.fields.Boolean`

    .. attribute:: sequence

        Alterable ordering criteria, allows drag-and-drop reordering of models
        in list views

        :type: :class:`~odoo.fields.Integer`

    .. attribute:: state

        lifecycle stages of the object, used by the ``states`` attribute on
        :class:`fields <odoo.fields.Field>`

        :type: :class:`~odoo.fields.Selection`

    .. attribute:: parent_id

        used to order records in a tree structure and enables the ``child_of``
        operator in domains

        :type: :class:`~odoo.fields.Many2one`

    .. attribute:: parent_left

        used with :attr:`~._parent_store`, allows faster tree structure access

    .. attribute:: parent_right

        see :attr:`~.parent_left`

.. _reference/orm/decorators:

Method decorators
=================

装饰器方法
=================

.. automodule:: odoo.api
    :members: multi, model, depends, constrains, onchange, returns,
              one, v7, v8

.. _reference/orm/fields:

字段
======

.. _reference/orm/fields/basic:

基础字段
------------

.. autodoc documents descriptors as attributes, even for the *definition* of
   descriptors. As a result automodule:: odoo.fields lists all the field
   classes as attributes without providing inheritance info or methods (though
   we don't document methods as they're not useful for "external" devs)
   (because we don't support pluggable field types) (or do we?)

.. autoclass:: odoo.fields.Field

.. autoclass:: odoo.fields.Char
    :show-inheritance:

.. autoclass:: odoo.fields.Boolean
    :show-inheritance:

.. autoclass:: odoo.fields.Integer
    :show-inheritance:

.. autoclass:: odoo.fields.Float
    :show-inheritance:

.. autoclass:: odoo.fields.Text
    :show-inheritance:

.. autoclass:: odoo.fields.Selection
    :show-inheritance:

.. autoclass:: odoo.fields.Html
    :show-inheritance:

.. autoclass:: odoo.fields.Date
    :show-inheritance:
    :members: today, context_today, from_string, to_string

.. autoclass:: odoo.fields.Datetime
    :show-inheritance:
    :members: now, context_timestamp, from_string, to_string

.. _reference/orm/fields/relational:

Relational fields
-----------------

.. autoclass:: odoo.fields.Many2one
    :show-inheritance:

.. autoclass:: odoo.fields.One2many
    :show-inheritance:

.. autoclass:: odoo.fields.Many2many
    :show-inheritance:

.. autoclass:: odoo.fields.Reference
    :show-inheritance:

.. _reference/orm/inheritance:

Inheritance and extension
=========================

Odoo provides three different mechanisms to extend models in a modular way:

Odoo提供三种不同以模块化扩展模块的机制

* creating a new model from an existing one, adding new information to the
  copy but leaving the original module as-is
* extending models defined in other modules in-place, replacing the previous
  version
* delegating some of the model's fields to records it contains

.. image:: ../images/inheritance_methods.png
    :align: center

Classical inheritance
---------------------

When using the :attr:`~odoo.models.Model._inherit` and
:attr:`~odoo.models.Model._name` attributes together, Odoo creates a new
model using the existing one (provided via
:attr:`~odoo.models.Model._inherit`) as a base. The new model gets all the
fields, methods and meta-information (defaults & al) from its base.

.. literalinclude:: ../../odoo/addons/test_documentation_examples/inheritance.py
    :language: python
    :lines: 5-

and using them:

.. literalinclude:: ../../odoo/addons/test_documentation_examples/tests/test_inheritance.py
    :language: python
    :lines: 8,12,9,19

will yield:

.. literalinclude:: ../../odoo/addons/test_documentation_examples/tests/test_inheritance.py
    :language: text
    :lines: 15,22

the second model has inherited from the first model's ``check`` method and its
``name`` field, but overridden the ``call`` method, as when using standard
:ref:`Python inheritance <python:tut-inheritance>`.

Extension
---------

When using :attr:`~odoo.models.Model._inherit` but leaving out
:attr:`~odoo.models.Model._name`, the new model replaces the existing one,
essentially extending it in-place. This is useful to add new fields or methods
to existing models (created in other modules), or to customize or reconfigure
them (e.g. to change their default sort order):

.. literalinclude:: ../../odoo/addons/test_documentation_examples/extension.py
    :language: python
    :lines: 5-

.. literalinclude:: ../../odoo/addons/test_documentation_examples/tests/test_extension.py
    :language: python
    :lines: 8,13

will yield:

.. literalinclude:: ../../odoo/addons/test_documentation_examples/tests/test_extension.py
    :language: text
    :lines: 11

.. note:: it will also yield the various :ref:`automatic fields
          <reference/orm/model/automatic>` unless they've been disabled

Delegation
----------

The third inheritance mechanism provides more flexibility (it can be altered
at runtime) but less power: using the :attr:`~odoo.models.Model._inherits`
a model *delegates* the lookup of any field not found on the current model
to "children" models. The delegation is performed via
:class:`~odoo.fields.Reference` fields automatically set up on the parent
model:

.. literalinclude:: ../../odoo/addons/test_documentation_examples/delegation.py
    :language: python
    :lines: 5-

.. literalinclude:: ../../odoo/addons/test_documentation_examples/tests/test_delegation.py
    :language: python
    :lines: 9-12,21,26

will result in:

.. literalinclude:: ../../odoo/addons/test_documentation_examples/tests/test_delegation.py
    :language: text
    :lines: 23,28

and it's possible to write directly on the delegated field:

.. literalinclude:: ../../odoo/addons/test_documentation_examples/tests/test_delegation.py
    :language: python
    :lines: 47

.. warning:: when using delegation inheritance, methods are *not* inherited,
             only fields

.. _reference/orm/domains:

Domains
=======

A domain is a list of criteria, each criterion being a triple (either a
``list`` or a ``tuple``) of ``(field_name, operator, value)`` where:

``field_name`` (``str``)
    a field name of the current model, or a relationship traversal through
    a :class:`~odoo.fields.Many2one` using dot-notation e.g. ``'street'``
    or ``'partner_id.country'``
``operator`` (``str``)
    an operator used to compare the ``field_name`` with the ``value``. Valid
    operators are:

    ``=``
        equals to
    ``!=``
        not equals to
    ``>``
        greater than
    ``>=``
        greater than or equal to
    ``<``
        less than
    ``<=``
        less than or equal to
    ``=?``
        unset or equals to (returns true if ``value`` is either ``None`` or
        ``False``, otherwise behaves like ``=``)
    ``=like``
        matches ``field_name`` against the ``value`` pattern. An underscore
        ``_`` in the pattern stands for (matches) any single character; a
        percent sign ``%`` matches any string of zero or more characters.
    ``like``
        matches ``field_name`` against the ``%value%`` pattern. Similar to
        ``=like`` but wraps ``value`` with '%' before matching
    ``not like``
        doesn't match against the ``%value%`` pattern
    ``ilike``
        case insensitive ``like``
    ``not ilike``
        case insensitive ``not like``
    ``=ilike``
        case insensitive ``=like``
    ``in``
        is equal to any of the items from ``value``, ``value`` should be a
        list of items
    ``not in``
        is unequal to all of the items from ``value``
    ``child_of``
        is a child (descendant) of a ``value`` record.

        Takes the semantics of the model into account (i.e following the
        relationship field named by
        :attr:`~odoo.models.Model._parent_name`).

``value``
    variable type, must be comparable (through ``operator``) to the named
    field

Domain criteria can be combined using logical operators in *prefix* form:

``'&'``
    logical *AND*, default operation to combine criteria following one
    another. Arity 2 (uses the next 2 criteria or combinations).
``'|'``
    logical *OR*, arity 2.
``'!'``
    logical *NOT*, arity 1.

    .. tip:: Mostly to negate combinations of criteria
        :class: aphorism

        Individual criterion generally have a negative form (e.g. ``=`` ->
        ``!=``, ``<`` -> ``>=``) which is simpler than negating the positive.

.. admonition:: Example

    To search for partners named *ABC*, from belgium or germany, whose language
    is not english::

        [('name','=','ABC'),
         ('language.code','!=','en_US'),
         '|',('country_id.code','=','be'),
             ('country_id.code','=','de')]

    This domain is interpreted as:

    .. code-block:: text

            (name is 'ABC')
        AND (language is NOT english)
        AND (country is Belgium OR Germany)

Porting from the old API to the new API
=======================================

* bare lists of ids are to be avoided in the new API, use recordsets instead
* methods still written in the old API should be automatically bridged by the
  ORM, no need to switch to the old API, just call them as if they were a new
  API method. See :ref:`reference/orm/oldapi/bridging` for more details.
* :meth:`~odoo.models.Model.search` returns a recordset, no point in e.g.
  browsing its result
* ``fields.related`` and ``fields.function`` are replaced by using a normal
  field type with either a ``related=`` or a ``compute=`` parameter
* :func:`~odoo.api.depends` on ``compute=`` methods **must be complete**,
  it must list **all** the fields and sub-fields which the compute method
  uses. It is better to have too many dependencies (will recompute the field
  in cases where that is not needed) than not enough (will forget to recompute
  the field and then values will be incorrect)
* **remove** all ``onchange`` methods on computed fields. Computed fields are
  automatically re-computed when one of their dependencies is changed, and
  that is used to auto-generate ``onchange`` by the client
* the decorators :func:`~odoo.api.model` and :func:`~odoo.api.multi` are
  for bridging *when calling from the old API context*, for internal or pure
  new-api (e.g. compute) they are useless
* remove :attr:`~odoo.models.Model._default`, replace by ``default=``
  parameter on corresponding fields
* if a field's ``string=`` is the titlecased version of the field name::

    name = fields.Char(string="Name")

  it is useless and should be removed
* the ``multi=`` parameter does not do anything on new API fields use the same
  ``compute=`` methods on all relevant fields for the same result
* provide ``compute=``, ``inverse=`` and ``search=`` methods by name (as a
  string), this makes them overridable (removes the need for an intermediate
  "trampoline" function)
* double check that all fields and methods have different names, there is no
  warning in case of collision (because Python handles it before Odoo sees
  anything)
* the normal new-api import is ``from odoo import fields, models``. If
  compatibility decorators are necessary, use ``from odoo import api,
  fields, models``
* avoid the :func:`~odoo.api.one` decorator, it probably does not do what
  you expect
* remove explicit definition of :attr:`~odoo.models.Model.create_uid`,
  :attr:`~odoo.models.Model.create_date`,
  :attr:`~odoo.models.Model.write_uid` and
  :attr:`~odoo.models.Model.write_date` fields: they are now created as
  regular "legitimate" fields, and can be read and written like any other
  field out-of-the-box
* when straight conversion is impossible (semantics can not be bridged) or the
  "old API" version is not desirable and could be improved for the new API, it
  is possible to use completely different "old API" and "new API"
  implementations for the same method name using :func:`~odoo.api.v7` and
  :func:`~odoo.api.v8`. The method should first be defined using the
  old-API style and decorated with :func:`~odoo.api.v7`, it should then be
  re-defined using the exact same name but the new-API style and decorated
  with :func:`~odoo.api.v8`. Calls from an old-API context will be
  dispatched to the first implementation and calls from a new-API context will
  be dispatched to the second implementation. One implementation can call (and
  frequently does) call the other by switching context.

  .. danger:: using these decorators makes methods extremely difficult to
              override and harder to understand and document
* uses of :attr:`~odoo.models.Model._columns` or
  :attr:`~odoo.models.Model._all_columns` should be replaced by
  :attr:`~odoo.models.Model._fields`, which provides access to instances of
  new-style :class:`odoo.fields.Field` instances (rather than old-style
  :class:`odoo.osv.fields._column`).

  Non-stored computed fields created using the new API style are *not*
  available in :attr:`~odoo.models.Model._columns` and can only be
  inspected through :attr:`~odoo.models.Model._fields`
* reassigning ``self`` in a method is probably unnecessary and may break
  translation introspection
* :class:`~odoo.api.Environment` objects rely on some threadlocal state,
  which has to be set up before using them. It is necessary to do so using the
  :meth:`odoo.api.Environment.manage` context manager when trying to use
  the new API in contexts where it hasn't been set up yet, such as new threads
  or a Python interactive environment::

    >>> from odoo import api, modules
    >>> r = modules.registry.RegistryManager.get('test')
    >>> cr = r.cursor()
    >>> env = api.Environment(cr, 1, {})
    Traceback (most recent call last):
      ...
    AttributeError: environments
    >>> with api.Environment.manage():
    ...     env = api.Environment(cr, 1, {})
    ...     print env['res.partner'].browse(1)
    ...
    res.partner(1,)

.. _reference/orm/oldapi/bridging:

Automatic bridging of old API methods
-------------------------------------

When models are initialized, all methods are automatically scanned and bridged
if they look like models declared in the old API style. This bridging makes
them transparently callable from new-API-style methods.

Methods are matched as "old-API style" if their second positional parameter
(after ``self``) is called either ``cr`` or ``cursor``. The system also
recognizes the third positional parameter being called ``uid`` or ``user`` and
the fourth being called ``id`` or ``ids``. It also recognizes the presence of
any parameter called ``context``.

When calling such methods from a new API context, the system will
automatically fill matched parameters from the current
:class:`~odoo.api.Environment` (for :attr:`~odoo.api.Environment.cr`,
:attr:`~odoo.api.Environment.user` and
:attr:`~odoo.api.Environment.context`) or the current recordset (for ``id``
and ``ids``).

In the rare cases where it is necessary, the bridging can be customized by
decorating the old-style method:

* disabling it entirely, by decorating a method with
  :func:`~odoo.api.noguess` there will be no bridging and methods will be
  called the exact same way from the new and old API styles
* defining the bridge explicitly, this is mostly for methods which are matched
  incorrectly (because parameters are named in unexpected ways):

  :func:`~odoo.api.cr`
     will automatically prepend the current cursor to explicitly provided
     parameters, positionally
  :func:`~odoo.api.cr_uid`
     will automatically prepend the current cursor and user's id to explictly
     provided parameters
  :func:`~odoo.api.cr_uid_ids`
     will automatically prepend the current cursor, user's id and recordset's
     ids to explicitly provided parameters
  :func:`~odoo.api.cr_uid_id`
     will loop over the current recordset and call the method once for each
     record, prepending the current cursor, user's id and record's id to
     explicitly provided parameters.

     .. danger:: the result of this wrapper is *always a list* when calling
                 from a new-API context

  All of these methods have a ``_context``-suffixed version
  (e.g. :func:`~odoo.api.cr_uid_context`) which also passes the current
  context *by keyword*.
* dual implementations using :func:`~odoo.api.v7` and
  :func:`~odoo.api.v8` will be ignored as they provide their own "bridging"

