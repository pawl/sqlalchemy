==============================
What's New in SQLAlchemy 1.2?
==============================

.. admonition:: About this Document

    This document describes changes between SQLAlchemy version 1.1
    and SQLAlchemy version 1.2.   1.2 is currently under development
    and is unreleased.


Introduction
============

This guide introduces what's new in SQLAlchemy version 1.2,
and also documents changes which affect users migrating
their applications from the 1.1 series of SQLAlchemy to 1.2.

Please carefully review the sections on behavioral changes for
potentially backwards-incompatible changes in behavior.

Platform Support
================

Targeting Python 2.7 and Up
---------------------------

SQLAlchemy 1.2 now moves the minimum Python version to 2.7, no longer
supporting 2.6.   New language features are expected to be merged
into the 1.2 series that were not supported in Python 2.6.  For Python 3 support,
SQLAlchemy is currnetly tested on versions 3.5 and 3.6.


New Features and Improvements - ORM
===================================

.. _change_3896_validates:

A @validates method receives all values on bulk-collection set before comparison
--------------------------------------------------------------------------------

A method that uses ``@validates`` will now receive all members of a collection
during a "bulk set" operation, before comparison is applied against the
existing collection.

Given a mapping as::

    class A(Base):
        __tablename__ = 'a'
        id = Column(Integer, primary_key=True)
        bs = relationship("B")

        @validates('bs')
        def convert_dict_to_b(self, key, value):
            return B(data=value['data'])

    class B(Base):
        __tablename__ = 'b'
        id = Column(Integer, primary_key=True)
        a_id = Column(ForeignKey('a.id'))
        data = Column(String)

Above, we could use the validator as follows, to convert from an incoming
dictionary to an instance of ``B`` upon collection append::

    a1 = A()
    a1.bs.append({"data": "b1"})

However, a collection assignment would fail, since the ORM would assume
incoming objects are already instances of ``B`` as it attempts to compare  them
to the existing members of the collection, before doing collection appends
which actually invoke the validator.  This would make it impossible for bulk
set operations to accomodate non-ORM objects like dictionaries that needed
up-front modification::

    a1 = A()
    a1.bs = [{"data": "b1"}]

The new logic uses the new :meth:`.AttributeEvents.bulk_replace` event to ensure
that all values are sent to the ``@validates`` function up front.

As part of this change, this means that validators will now receive
**all** members of a collection upon bulk set, not just the members that
are new.   Supposing a simple validator such as::

    class A(Base):
        # ...

        @validates('bs')
        def validate_b(self, key, value):
            assert value.data is not None
            return value

Above, if we began with a collection as::

    a1 = A()

    b1, b2 = B(data="one"), B(data="two")
    a1.bs = [b1, b2]

And then, replaced the collection with one that overlaps the first::

    b3 = B(data="three")
    a1.bs = [b2, b3]

Previously, the second assignment would trigger the ``A.validate_b``
method only once, for the ``b3`` object.  The ``b2`` object would be seen
as being already present in the collection and not validated.  With the new
behavior, both ``b2`` and ``b3`` are passed to ``A.validate_b`` before passing
onto the collection.   It is thus important that valiation methods employ
idempotent behavior to suit such a case.

.. seealso::

    :ref:`change_3896_event`

:ticket:`3896`

.. _change_3896_event:

New bulk_replace event
----------------------

To suit the validation use case described in :ref:`change_3896_validates`,
a new :meth:`.AttributeEvents.bulk_replace` method is added, which is
called in conjunction with the :meth:`.AttributeEvents.append` and
:meth:`.AttributeEvents.remove` events.  "bulk_replace" is called before
"append" and "remove" so that the collection can be modified ahead of comparison
to the existing collection.   After that, individual items
are appended to a new target collection, firing off the "append"
event for items new to the collection, as was the previous behavior.
Below illustrates both "bulk_replace" and
"append" at the same time, including that "append" will receive an object
already handled by "bulk_replace" if collection assignment is used.
A new symbol :attr:`~.attributes.OP_BULK_REPLACE` may be used to determine
if this "append" event is the second part of a bulk replace::

    from sqlalchemy.orm.attributes import OP_BULK_REPLACE

    @event.listens_for(SomeObject.collection, "bulk_replace")
    def process_collection(target, values, initiator):
        values[:] = [_make_value(value) for value in values]

    @event.listens_for(SomeObject.collection, "append", retval=True)
    def process_collection(target, value, initiator):
        # make sure bulk_replace didn't already do it
        if initiator is None or initiator.op is not OP_BULK_REPLACE:
            return _make_value(value)
        else:
            return value


:ticket:`3896`


New Features and Improvements - Core
====================================

.. _change_2694:

New "autoescape" option for startswith(), endswith()
----------------------------------------------------

The "autoescape" parameter is added to :meth:`.Operators.startswith`,
:meth:`.Operators.endswith`, :meth:`.Operators.contains`.  This parameter
does what "escape" does, except that it also automatically performs a search-
and-replace of any wildcard characters to be escaped by that character, as
these operators already add the wildcard expression on the outside of the
given value.

An expression such as::

    >>> column('x').startswith('total%score', autoescape='/')

Renders as::

    x LIKE :x_1 || '%%' ESCAPE '/'

Where the value of the parameter "x_1" is ``'total/%score'``.

:ticket:`2694`

Key Behavioral Changes - ORM
============================

.. _change_3934:

The after_rollback() Session event now emits before the expiration of objects
-----------------------------------------------------------------------------

The :meth:`.Session.after_rollback` event now has access to the attribute
state of objects before their state has been expired (e.g. the "snapshot
removal").  This allows the event to be consistent with the behavior
of the :meth:`.Session.after_commit` event which also emits before the
"snapshot" has been removed::

    sess = Session()

    user = sess.query(User).filter_by(name='x').first()

    @event.listens_for(sess, "after_rollback")
    def after_rollback(session):
        # 'user.name' is now present, assuming it was already
        # loaded.  previously this would raise upon trying
        # to emit a lazy load.
        print("user name: %s" % user.name)

    @event.listens_for(sess, "after_commit")
    def after_commit(session):
        # 'user.name' is present, assuming it was already
        # loaded.  this is the existing behavior.
        print("user name: %s" % user.name)

    if should_rollback:
        sess.rollback()
    else:
        sess.commit()

Note that the :class:`.Session` will still disallow SQL from being emitted
within this event; meaning that unloaded attributes will still not be
able to load within the scope of the event.

:ticket:`3934`

.. _change_3891:

Fixed issue involving single-table inheritance with ``select_from()``
---------------------------------------------------------------------

The :meth:`.Query.select_from` method now honors the single-table inheritance
column discriminator when generating SQL; previously, only the expressions
in the query column list would be taken into account.

Supposing ``Manager`` is a subclass of ``Employee``.  A query like the following::

    sess.query(Manager.id)

Would generate SQL as::

    SELECT employee.id FROM employee WHERE employee.type IN ('manager')

However, if ``Manager`` were only specified by :meth:`.Query.select_from`
and not in the columns list, the discriminator would not be added::

    sess.query(func.count(1)).select_from(Manager)

would generate::

    SELECT count(1) FROM employee

With the fix, :meth:`.Query.select_from` now works correctly and we get::

    SELECT count(1) FROM employee WHERE employee.type IN ('manager')

Applications that may have been working around this by supplying the
WHERE clause manually may need to be adjusted.

:ticket:`3891`

.. _change_3913:

Previous collection is no longer mutated upon replacement
---------------------------------------------------------

The ORM emits events whenever the members of a mapped collection change.
In the case of assigning a collection to an attribute that would replace
the previous collection, a side effect of this was that the collection
being replaced would also be mutated, which is misleading and unnecessary::

    >>> a1, a2, a3 = Address('a1'), Address('a2'), Address('a3')
    >>> user.addresses = [a1, a2]

    >>> previous_collection = user.addresses

    # replace the collection with a new one
    >>> user.addresses = [a2, a3]

    >>> previous_collection
    [Address('a1'), Address('a2')]

Above, prior to the change, the ``previous_collection`` would have had the
"a1" member removed, corresponding to the member that's no longer in the
new collection.

:ticket:`3913`

Key Behavioral Changes - Core
=============================

.. _change_3907:

The IN / NOT IN operators render a simplified boolean expression with an empty collection
-----------------------------------------------------------------------------------------

An expression such as ``column.in_([])``, which is assumed to be false,
now produces the expression ``1 != 1``
by default, instead of ``column != column``.  This will **change the result**
of a query that is comparing a SQL expression or column that evaluates to
NULL when compared to an empty set, producing a boolean value false or true
(for NOT IN) rather than NULL.  The warning that would emit under
this condition is also removed.  The old behavior is available using the
:paramref:`.create_engine.empty_in_strategy` parameter to
:func:`.create_engine`.

In SQL, the IN and NOT IN operators do not support comparison to a
collection of values that is explicitly empty; meaning, this syntax is
illegal::

    mycolumn IN ()

To work around this, SQLAlchemy and other database libraries detect this
condition and render an alternative expression that evaluates to false, or
in the case of NOT IN, to true, based on the theory that "col IN ()" is always
false since nothing is in "the empty set".    Typically, in order to
produce a false/true constant that is portable across databases and works
in the context of the WHERE clause, a simple tautology such as ``1 != 1`` is
used to evaluate to false and ``1 = 1`` to evaluate to true (a simple constant
"0" or "1" often does not work as the target of a WHERE clause).

SQLAlchemy in its early days began with this approach as well, but soon it
was theorized that the SQL expression ``column IN ()`` would not evaluate to
false if the "column" were NULL; instead, the expression would produce NULL,
since "NULL" means "unknown", and comparisons to NULL in SQL usually produce
NULL.

To simulate this result, SQLAlchemy changed from using ``1 != 1`` to
instead use th expression ``expr != expr`` for empty "IN" and ``expr = expr``
for empty "NOT IN"; that is, instead of using a fixed value we use the
actual left-hand side of the expression.  If the left-hand side of
the expression passed evaluates to NULL, then the comparison overall
also gets the NULL result instead of false or true.

Unfortunately, users eventually complained that this expression had a very
severe performance impact on some query planners.   At that point, a warning
was added when an empty IN expression was encountered, favoring that SQLAlchemy
continues to be "correct" and urging users to avoid code that generates empty
IN predicates in general, since typically they can be safely omitted.  However,
this is of course burdensome in the case of queries that are built up dynamically
from input variables, where an incoming set of values might be empty.

In recent months, the original assumptions of this decision have been
questioned.  The notion that the expression "NULL IN ()" should return NULL was
only theoretical, and could not be tested since databases don't support that
syntax.  However, as it turns out, you can in fact ask a relational database
what value it would return for "NULL IN ()" by simulating the empty set as
follows::

    SELECT NULL IN (SELECT 1 WHERE 1 != 1)

With the above test, we see that the databases themselves can't agree on
the answer.  Postgresql, considered by most to be the most "correct" database,
returns False; because even though "NULL" represents "unknown", the "empty set"
means nothing is present, including all unknown values.  On the
other hand, MySQL and MariaDB return NULL for the above expression, defaulting
to the more common behavior of "all comparisons to NULL return NULL".

SQLAlchemy's SQL architecture is more sophisticated than it was when this
design decision was first made, so we can now allow either behavior to
be invoked at SQL string compilation time.  Previously, the conversion to a
comparison expression were done at construction time, that is, the moment
the :meth:`.ColumnOperators.in_` or :meth:`.ColumnOperators.notin_` operators were invoked.
With the compilation-time behavior, the dialect itself can be instructed
to invoke either approach, that is, the "static" ``1 != 1`` comparison or the
"dynamic" ``expr != expr`` comparison.   The default has been **changed**
to be the "static" comparison, since this agrees with the behavior that
Postgresql would have in any case and this is also what the vast majority
of users prefer.   This will **change the result** of a query that is comparing
a null expression to the empty set, particularly one that is querying
for the negation ``where(~null_expr.in_([]))``, since this now evaluates to true
and not NULL.

The behavior can now be controlled using the flag
:paramref:`.create_engine.empty_in_strategy`, which defaults to the
``"static"`` setting, but may also be set to ``"dynamic"`` or
``"dynamic_warn"``, where the ``"dynamic_warn"`` setting is equivalent to the
previous behavior of emitting ``expr != expr`` as well as a performance
warning.   However, it is anticipated that most users will appreciate the
"static" default.

:ticket:`3907`

Dialect Improvements and Changes - PostgreSQL
=============================================

Dialect Improvements and Changes - MySQL
=============================================

Dialect Improvements and Changes - SQLite
=============================================

Dialect Improvements and Changes - Oracle
=============================================

.. _change_3276:

Oracle foreign key constraint names are now "name normalized"
-------------------------------------------------------------

The names of foreign key constraints as delivered to a
:class:`.ForeignKeyConstraint` object during table reflection as well as
within the :meth:`.Inspector.get_foreign_keys` method will now be
"name normalized", that is, expressed as lower case for a case insensitive
name, rather than the raw UPPERCASE format that Oracle uses::

    >>> insp.get_indexes("addresses")
    [{'unique': False, 'column_names': [u'user_id'],
      'name': u'address_idx', 'dialect_options': {}}]

    >>> insp.get_pk_constraint("addresses")
    {'name': u'pk_cons', 'constrained_columns': [u'id']}

    >>> insp.get_foreign_keys("addresses")
    [{'referred_table': u'users', 'referred_columns': [u'id'],
      'referred_schema': None, 'name': u'user_id_fk',
      'constrained_columns': [u'user_id']}]

Previously, the foreign keys result would look like::

    [{'referred_table': u'users', 'referred_columns': [u'id'],
      'referred_schema': None, 'name': 'USER_ID_FK',
      'constrained_columns': [u'user_id']}]

Where the above could create problems particularly with Alembic autogenerate.

:ticket:`3276`

