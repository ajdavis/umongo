.. _userguide:

==========
User guide
==========

Base concepts
=============


In μMongo 3 worlds are considered:

.. figure:: data_flow.png
   :alt: data flow in μMongo


Client world
------------

This is the data from outside μMongo, it can be JSON dict from your web framework
(i.g. ``request.get_json()`` with `flask <http://flask.pocoo.org/>`_ or
``json.loads(request.raw_post_data)`` in `django <https://www.djangoproject.com/>`_)
or it could be regular Python dict with Python-typed data

JSON dict example

.. code-block:: python

    >>> {"str_field": "hello world", "int_field": 42, "date_field": "2015-01-01T00:00:00Z"}

Python dict example

.. code-block:: python

    >>> {"str_field": "hello world", "int_field": 42, "date_field": datetime(2015, 1, 1)}

To be integrated into μMongo, those data need to be unserialized. Same thing
to leave μMongo they need to be serialized (under the hood
μMongo uses `marshmallow <http://marshmallow.readthedocs.org/>`_ schema).
The unserialization operation is done automatically when instantiating a
:class:`umongo.Document`. The serialization is done when calling
:meth:`umongo.Document.dump` on a document instance.


Object Oriented world
---------------------

So what's good about :class:`umongo.Document` ? Well it allows you to work
with your data as Objects and to guarantee there validity against a model.

First let's define a document with few :mod:`umongo.fields`

.. code-block:: python

    class Dog(Document):
        name = fields.StrField(required=True)
        breed = fields.StrField(missing="Mongrel")
        birthday = fields.DateTimeField()

Note that each field can be customized with special attributes like
``required`` (which is pretty self-explanatory) or ``missing`` (if the
field is missing during unserialization it will take this value).

Now we can play back and forth between OO and client worlds

.. code-block:: python

    >>> client_data = {'name': 'Odwin', 'birthday': '2001-09-22T00:00:00Z'}
    >>> odwin = Dog(**client_data)
    >>> odwin.breed
    "Mongrel"
    >>> odwin.birthday
    datetime.datetime(2001, 9, 22, 0, 0)
    >>> odwin.breed = "Labrador"
    >>> odwin.dump()
    {'birthday': '2001-09-22T00:00:00+00:00', 'breed': 'Labrador', 'name': 'Odwin'}

.. note:: You can access the data as attribute (i.g. ``odwin.name``) or as item (i.g. ``odwin['name']``).
          The latter is specially useful if one of your field name clash with :class:`umongo.Document`'s attributes.

OO world enforces model validation for each modification

.. code-block:: python

    >>> odwin.bad_field = 42
    [...]
    AttributeError: bad_field
    >>> odwin.birthday = "not_a_date"
    [...]
    ValidationError: Not a valid datetime.

.. note: Just one exception: ``required`` attribute is validate at insertion time, we'll talk about that later.

Object orientation means inheritance, of course you can do that

.. code-block:: python

    class Animal(Document):
        breed = fields.StrField()
        birthday = fields.DateTimeField()

        class Meta:
            allow_inheritance = True
            abstract = True

    class Dog(Animal):
        name = fields.StrField(required=True)

    class Duck(Animal):
        pass

Note the ``Meta`` subclass, it is used (along with inherited Meta classes from
parent documents) to configure the document class, you can access this final
config through the ``opts`` attribute.

Here we use this to allow ``Animal`` to be inheritable and to make it abstract.

.. code-block:: python

    >>> Animal.opts
    <DocumentOpts(abstract=True, allow_inheritance=True, is_child=False, base_schema_cls=<class 'umongo.schema.Schema'>, indexes=[], custom_indexes=[], collection=None, lazy_collection=None, dal=None, children={'Duck', 'Dog'})>
    >>> Dog.opts
    <DocumentOpts(abstract=False, allow_inheritance=False, is_child=False, base_schema_cls=<class 'umongo.schema.Schema'>, indexes=[], custom_indexes=[], collection=None, lazy_collection=None, dal=None, children={})>
    >>> class NotAllowedSubDog(Dog): pass
    [...]
    DocumentDefinitionError: Document <class '__main__.Dog'> doesn't allow inheritance
    >>> Animal(breed="Mutant")
    [...]
    AbstractDocumentError: Cannot instantiate an abstract Document



Mongo world
-----------

What the point of a MongoDB ODM without MongoDB ? So here it is !

Mongo world consist of data returned in a format comprehensible by a MongoDB
driver (`pymongo <https://api.mongodb.org/python/current/>`_ for instance).

.. code-block:: python

    >>> odwin.to_mongo()
    {'birthday': datetime.datetime(2001, 9, 22, 0, 0), 'name': 'Odwin'}

Well it our case the data haven't change much (if any !). Let's consider something more complex:

.. code-block:: python

    class Dog(Document):
        name = fields.StrField(attribute='_id')

Here we decided to use the name of the dog as our ``_id`` key, but for
readability we keep it as ``name`` inside our document.

.. code-block:: python

    >>> odwin = Dog(name='Odwin')
    >>> odwin.dump()
    {'name': 'Odwin'}
    >>> odwin.to_mongo()
    {'_id': 'Odwin'}
    >>> Dog.build_from_mongo({'_id': 'Scruffy'}).dump()
    {'name': 'Scruffy'}

.. note:: If no field refers to ``_id`` in the document, a dump-only field ``id``
          will be automatically added:

          .. code-block:: python

            >>> class AutoId(Document):
            ...     pass
            >>> AutoId.find_one()
            <object Document __main__.AutoId({'_id': ObjectId('5714b9a61d41c8feb01222c8')})>

But what about if we what to retrieve the ``_id`` field whatever it name is ?
No problem, use the ``pk`` property:

.. code-block:: python

    >>> odwin.pk
    'Odwin'
    >>> Duck().pk
    None

Ok so now we got our data in a way we can insert it to MongoDB through our favorite driver.
In fact most of the time you don't need to use ``to_mongo`` directly.
Instead you should configure (remember the ``Meta`` class ?) you document class
with a collection to insert into:

.. code-block:: python

    >>> db = pymongo.MongoClient().umongo_test
    >>> class Dog(Document):
    ...     name = fields.StrField(attribute='_id')
    ...     breed = fields.StrField(missing="Mongrel")
    ...     class Meta:
    ...         collection = db.dog

.. note::
    Often in more complex applications you won't have your driver ready
    when defining your documents. In such case you should use instead ``lazy_collection``
    with a lazy loader depending of your driver:

    .. code-block:: python

          def get_collection():
              return txmongo.MongoConnection()['lazy_db_doc']

          class LazyDBDoc(Document):
              class Meta:
                  lazy_collection = txmongo_lazy_loader(get_collection)


This way you will be able to ``commit`` your changes into the database:

.. code-block:: python

    >>> odwin = Dog(name='Odwin', breed='Labrador')
    >>> odwin.commit()

You get also access to Object Oriented version of your driver methods:

.. code-block:: python

    >>> Dog.find()
    <umongo.dal.pymongo.WrappedCursor object at 0x7f169851ba68>
    >>> next(Dog.find())
    <object Document __main__.Dog({'_id': 'Odwin', 'breed': 'Labrador'})>
    Dog.find_one({'_id': 'Odwin'})
    <object Document __main__.Dog({'_id': 'Odwin', 'breed': 'Labrador'})>


Multi-driver support
====================

For the moment all examples has been done with pymongo, but thing are
pretty the same with other drivers, just change the ``collection`` and you're good to go:

.. code-block:: python

    >>> db = motor.motor_asyncio.AsyncIOMotorClient()['umongo_test']
    >>> class Dog(Document):
    ...     name = fields.StrField(attribute='_id')
    ...     breed = fields.StrField(missing="Mongrel")
    ...     class Meta:
    ...         collection = db.dog

Of course the way you'll be calling methods will differ:

.. code-block:: python

    >>> odwin = Dog(name='Odwin', breed='Labrador')
    >>> yield from odwin.commit()
    >>> dogs = yield from Dog.find()

.. note:: Be careful not to mix documents with different collection type
          defined or unexpected thing could happened (and furthermore there
          is no practical reason to do that !)


Inheritance
===========

Inheritance inside the same collection is achieve by adding a ``_cls`` field
(accessible in the document as ``cls``) in the document stored in MongoDB

.. code-block:: python

    >>> class Parent(Document):
    ...     unique_in_parent = fields.IntField(unique=True)
    ...     class Meta:
    ...         allow_inheritance = True
    >>> class Child(Parent):
    ...     unique_in_child = fields.StrField(unique=True)
    >>> child = Child(unique_in_parent=42, unique_in_child='forty_two')
    >>> child.cls
    'Child'
    >>> child.dump()
    {'cls': 'Child', 'unique_in_parent': 42, 'unique_in_child': 'forty_two'}
    >>> Parent().dump(unique_in_parent=22)
    {'unique_in_parent': 22}
    >>> [x.document for x in Parent.opts.indexes]
    [{'key': SON([('unique_in_parent', 1)]), 'name': 'unique_in_parent_1', 'sparse': True, 'unique': True}]


Indexes
=======

.. warning:: Indexes must be first submitted to MongoDB. To do so you should
             call :meth:`umongo.Document.ensure_indexes` once for each document


In fields, ``unique`` attribute is implicitly handled by an index:

.. code-block:: python

    >>> class WithUniqueEmail(Document):
    ...     email = fields.StrField(unique=True)
    >>> [x.document for x in WithUniqueEmail.opts.indexes]
    [{'key': SON([('email', 1)]), 'name': 'email_1', 'sparse': True, 'unique': True}]
    >>> WithUniqueEmail.ensure_indexes()
    >>> WithUniqueEmail().commit()
    >>> WithUniqueEmail().commit()
    [...]
    ValidationError: {'email': 'Field value must be unique'}

.. note:: The index params also depend of the ``required``, ``null`` field attributes

For more custom indexes, the ``Meta.indexes`` attribute should be used:

.. code-block:: python

    >>> class CustomIndexes(Document):
    ...     name = fields.StrField()
    ...     age = fields.Int()
    ...     class Meta:
    ...         indexes = ('#name', 'age', ('-age', 'name'))
    >>> [x.document for x in CustomIndexes.opts.indexes]
    [{'key': SON([('name', 'hashed')]), 'name': 'name_hashed'},
     {'key': SON([('age', 1), ]), 'name': 'age_1'},
     {'key': SON([('age', -1), ('name', 1)]), 'name': 'age_-1_name_1'}

.. note:: ``Meta.indexes`` should use the names of the fields as they appear
          in database (i.g. given a field ``nick = StrField(attribute='nk')``,
          you refer to it in ``Meta.indexes`` as ``nk``)

Indexes can be passed as:

- a string with an optional direction prefix (i.g. ``"my_field"``)
- a list of string with optional direction prefix for compound indexes
  (i.g. ``["field1", "-field2"]``)
- a :class:`pymongo.IndexModel` object
- a dict used to instantiate an :class:`pymongo.IndexModel` for custom configuration
  (i.g. ``{'key': ['field1', 'field2'], 'expireAfterSeconds': 42}``)

Allowed direction prefix are:
 - ``+`` for ascending
 - ``-`` for descending
 - ``$`` for text
 - ``#`` for hashed

.. note:: If no direction prefix is passed, ascending is assumed

In case of a field defined in a child document, it index is automatically
compounded with the ``_cls``

.. code-block:: python

      >>> class Parent(Document):
      ...     unique_in_parent = fields.IntField(unique=True)
      ...     class Meta:
      ...         allow_inheritance = True
      >>> class Child(Parent):
      ...     unique_in_child = fields.StrField(unique=True)
      ...     class Meta:
      ...         indexes = ['#unique_in_parent']
      >>> [x.document for x in Child.opts.indexes]
      [{'name': 'unique_in_parent_1', 'sparse': True, 'unique': True, 'key': SON([('unique_in_parent', 1)])},
       {'name': 'unique_in_parent_hashed__cls_1', 'key': SON([('unique_in_parent', 'hashed'), ('_cls', 1)])},
       {'name': '_cls_1', 'key': SON([('_cls', 1)])},
       {'name': 'unique_in_child_1__cls_1', 'sparse': True, 'unique': True, 'key': SON([('unique_in_child', 1), ('_cls', 1)])}]


I18n
====

μMongo provides a simple way to work with i18n (internationalization) through
the :func:`umongo.set_gettext`, for example to use python's default gettext:

.. code-block:: python

    from umongo import set_gettext
    from gettext import gettext
    set_gettext(gettext)

This way each error message will be passed to the custom ``gettext`` function
in order for it to return the localized version of it.

See `examples/flask <https://github.com/Scille/umongo/tree/master/examples/flask>`_
for a working example of i18n with `flask-babel <https://pythonhosted.org/Flask-Babel/>`_.

.. note::
    To set up i18n inside your app, you should start with `messages.pot
    <https://github.com/Scille/umongo/tree/master/messages.pot>`_ which is
    a translation template of all the messages used in umongo (and it dependancy marshmallow).


Marshmallow integration
=======================

Under the hood, μMongo heavily uses `marshmallow <http://marshmallow.readthedocs.org>`_
for all it data validation work.

However an ODM has some special needs (i.g. handling ``required`` fields that
are handled by MongoDB's unique indexes) that force to extend marshmallow base types.

In short, you should not try to use marshmallow base types (:class:`marshmallow.Schema`,
:class:`marshmallow.fields.Field` or :class:`marshmallow.validate.Validator` for instance)
in a μMongo document but instead use their μMongo equivalents (respectively
:class:`umongo.abstract.BaseSchema`, :class:`umongo.abstract.BaseField` and
:class:`umongo.abstract.BaseValidator`).


Field validate & io_validate
============================

Fields can be configured with special validators through the ``validate`` attribute:

.. code-block:: python

    from umongo import Document, fields, validate

    class Employee(Document):
        name = fields.StrField(validate=[validate.Length(max=120), validate.Regexp(r"[a-zA-Z ']+")])
        age = fields.IntField(validate=validate.Range(min=18, max=65))
        email = fields.StrField(validate=validate.Email())
        type = fields.StrField(validate=validate.OneOf(['private', 'sergeant', 'general']))

Those validators will be enforced each time a field is modified:

.. code-block:: python

    >>> john = Employee(name='John Rambo')
    >>> john.age = 99  # it's not his war anymore...
    [...]
    ValidationError: {'age': ["No way I'm doing this !"]}

Now sometime you'll need for your validator to query your database (this
is mainly done to validate a :class:`umongo.data_objects.Reference`). For
this need you can use the ``io_validate`` attribute.
This attribute should get passed a function (or a list of functions) that
will do database access in accordance with the used mongodb driver.

For example with Motor-asyncio driver, ``io_validate``'s functions will be
wrapped by :class:`asyncio.coroutine` and called with ``yield from``.

.. code-block:: python

    from motor.motor_asyncio import AsyncIOMotorClient
    db = AsyncIOMotorClient().test


    class TrendyActivity(Document):
        name = fields.StrField()

        class Meta:
            collection = db.trendy_activity


    class Job(Document):

        def _is_dream_job(field, value):
            if not (yield from TrendyActivity.find_one(name=value)):
                raise ValidationError("No way I'm doing this !")

        activity = fields.StrField(io_validate=_is_dream_job)

        class Meta:
            collection = db.job

    @asyncio.coroutine
    def run():
        yield from TrendyActivity(name='Pythoning').commit()
        yield from Job(activity='Pythoning').commit()
        yield from Job(activity='Javascripting...').commit()
        # raises ValidationError: {'activity': ["No way I'm doing this !"]}
