..
        SPDX-License-Identifier: CC-BY-SA-4.0 or-later
        SPDX-FileCopyrightText: 2023 grommunio GmbH

PHP Environment Notes
=====================

calendar
--------

The ``calendar`` extension (php-calendar) defines ``CAL_GREGORIAN`` for the the
PHP environment, which clashes with equally-named defines in our
*mapi-php-header* package.

php-calender even eschews Windows's own definitions.

.. code-block:: c

   #ifdef PHP_WIN32
   /* This conflicts with a define in winnls.h, but that header is needed
      to have GetACP(). */
   #undef CAL_GREGORIAN
   #endif


opcache
-------

The ``opcache`` extension mis-executes the ZEND_TYPE_CHECK opcode in at least
PHP 8.0.25 and causes a vital PHP expression, ``is_resource($x)``, to
spuriously yield ``false`` even when ``$x`` is a valid resource. Spurious as
in: we do not know why, but it is reproducibly the same line in grommunio-web.

Details:

We find that the Zend engine treats the PHP ``is_resource(...)`` function call
specially[`↗
<https://github.com/php/php-src/blob/master/Zend/zend_compile.c#L4497>_`],

.. code-block:: c

	} else if (zend_string_equals_literal(lcname, "is_resource")) {
		return zend_compile_func_typecheck(result, args, IS_RESOURCE);

and that Zend compiles it to a ``ZEND_TYPE_CHECK`` opcode with extended_value
being ``(1 << IS_RESOURCE)``[`↗
<https://github.com/php/php-src/blob/master/Zend/zend_compile.c#L3945..L3950>`_].

.. code-block:: c

	opline = zend_emit_op_tmp(result, ZEND_TYPE_CHECK, &arg_node, NULL);
	if (type != _IS_BOOL) {
		opline->extended_value = (1 << type);
	} else {
		opline->extended_value = (1 << IS_FALSE) | (1 << IS_TRUE);
	}

When the two lines shown in the first block are removed, the
``is_resource($x)`` expression would instead be compiled to a
``ZEND_INIT_FCALL`` opcode[`↗
<https://github.com/php/php-src/blob/master/Zend/zend_compile.c#L4612>`_].
 and execution would eventually land in the C function corresponding to
``is_resource`` [`↗
<https://github.com/php/php-src/blob/php-8.0.25/ext/standard/type.c#L240..L276>`_].

.. code-block:: c

	static inline void php_is_type(INTERNAL_FUNCTION_PARAMETERS, int type)
	{
		if (Z_TYPE_P(arg) == type) {
		...

Summarizing our observations:

* Unmodified Zend VM, php-opcache disabled, ``is_resource`` becomes
  ``ZEND_TYPE_CHECK``: good
* Unmodified Zend VM, php-opcache enabled, ``is_resource`` becomes
  ``ZEND_TYPE_CHECK``: bad
* Modified Zend VM, php-opcache enabled, ``is_resource`` becomes
  ``ZEND_INIT_FCALL``: good
* We conclude that php-opcache induces a problem with respect to the
  ``ZEND_TYPE_CHECK`` opcode.

There is a... peculiar comment in php-opcache (``MAY_BE_RESOURCE`` is the same
as ``1 << IS_RESOURCE``)[↗
<https://github.com/php/php-src/blob/master/ext/opcache/jit/zend_jit.c#L3515>`_]
that could(?) be relevant:

.. code-block:: c

	case ZEND_TYPE_CHECK:
		if (opline->extended_value == MAY_BE_RESOURCE) {
			// TODO: support for is_resource() ???
			break;
		}
