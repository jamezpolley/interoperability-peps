PEP: 0496
Title: Environment Markers
Version: $Revision$
Last-Modified: $Date$
Author: James Polley <jp@jamezpolley.com>
BDFL-Delegate: Nick Coghlan <ncoghlan@gmail.com>
Status: Draft
Type: Informational
Content-Type: text/x-rst
Created: 02-Jul-2015
Post-History: 03-Jul-2015


Abstract
========

Python packages can have dependencies on other python packages. In
some cases, these dependencies are conditional on the environment into
which the package is being installed.

An **environment marker** describes a condition about the current
execution environment. They are used to indicate whether a particular
dependency will apply to a given environment.

Environment markers were first specified in PEP-0345[1]. PEP-0426[2] (which
would replace PEP-0345) proposed extensions to the markers. When
2.7.10 was released, even these extensions became insufficient due to
their reliance on simple lexical comparisons, and thus this PEP has
been born.

Rationale
=========

Many Python packages are written with portability in mind.

For many packages this means they aim to support a wide range of
Python releases. If they depend on libraries such as ``argparse`` -
which started as external libraries, but later got incorporated into
core - specifying a single set of requirements is difficult, as the
set of required packages differs depending on the version of Python in
use.

For other packages, designing for portability means supporting
multiple operating systems. However, the significant differences
between them may mean that particular dependencies are only needed on
particular platforms (relying on ``pywin32`` only on Windows, for
example)"

Environment Markers attempt to provide more flexibility in a list of
requirements by allowing the developer to list requirements that are
specific to a particular environment.

Alternatives considered
=======================

For a while, OpenStack used seperate requirements files for different
environments - eg, ``requirements-py2.txt`` and
``requirements-py3.txt``. However, in most cases the two files were
near duplicates, with only one or two lines different; but maintaining
two seperate files was more than twice as much work, as it required
manual checking to keep the files in sync.

Keeping all the requirements together in one place, with markers
indicating which to install, has been easier to maintain.

Examples
========

Here is an example of such markers inside a requirements.txt
file. This will install ``pywin32`` on Windows; ``unittest2`` on
python2.4 or python2.5; ``backports.ssl_match_hostname`` on the
versions where ssl_match_hostname is not already implemented; and
``hawkey`` only on linux::

   pywin32 >=1.0 ; sys_platform == 'win32'
   unittest2 >=2.0,<3.0 ; python_version == '2.4' or python_version == '2.5'
   backports.ssl_match_hostname >= 3.4 ; python_version < '3.4' and python_version != '2.7.10'
   hawkey ; 'linux' in sys_platform

And here's an example of some conditional metadata included in
setup.py for a distribution that requires PyWin32 both at runtime and
buildtime when using Windows::

   setup(
     install_requires=["pywin32 > 1.0 : sys.platform == 'win32'"],
     setup_requires=["pywin32 > 1.0 : sys.platform == 'win32'"]
     )


Micro-language
==============

The micro-language behind this is as follows. It compares:

* strings with the ``==`` and ``in`` operators (and their opposites)
* version numbers with the ``==``, ``!=``, ``<``, ``<=``, ``>=``, and ``<`` operators.


The usual boolean operators ``and`` and ``or`` can be used to combine
expressions, and parentheses are supported for grouping.

The pseudo-grammar is ::

    MARKER: EXPR [(and|or) EXPR]*
    EXPR: ("(" MARKER ")") | (STREXPR|VEREXPR)
    STREXPR: STRING [STRCMPOP STREXPR]
    STRCMPOP: ==|!=|in|not in
    VEREXPR: VERSION [VERCMPOP VEREXPR]
    VERCMPOP: (==|!=|<|>|<=|>=)


``SUBEXPR`` is either a Python string (such as ``'win32'``) or one of
the ``Strings`` marker variables listed below.

``VEREXPR`` is a PEP-0440[3] version identifier, or one of the
``Version number`` marker variables listed below. Comparisons between
version numbers are done using PEP-0440 semantics.


Strings
-------

.. list-table::
   :header-rows: 1

   * - Marker
     - Python equivalent
     - Sample values
   * - ``os_name``
     - ``os.name``
     - ``posix``, ``java``
   * - ``sys_platform``
     - ``sys.platform``
     - ``linux``, ``darwin``, ``java1.8.0_51``
   * - ``platform_release``
     - ``platform.release()``
     - ``3.14.1-x86_64-linode39``, ``14.5.0``, ``1.8.0_51``
   * - ``platform_machine``
     - ``platform.machine()``
     - ``x86_64``
   * - ``platform_python_implementation``
     - ``platform.python_implementation()``
     - ``CPython``, ``Jython``
   * - ``implementation_name``
     - ``sys.implementation.name``
     - ``cpython``
   * - ``platform_version``
     - ``platform.version()``
     - ``#1 SMP Fri Apr 25 13:07:35 EDT 2014``

       ``Java HotSpot(TM) 64-Bit Server VM, 25.51-b03, Oracle Corporation``

       ``Darwin Kernel Version 14.5.0: Wed Jul 29 02:18:53 PDT 2015; root:xnu-2782.40.9~2/RELEASE_X86_64``
   * - ``platform_dist_name``
     - ``platform.dist()[0]``
     - ``Ubuntu``
   * - ``platform_dist_version``
     - ``platform.dist()[1]``
     - ``14.04``
   * - ``platform_dist_id``
     - ``platform.dist()[2]``
     - ``trusty``

If a particular string value is not available (such as
``sys.implementation.name`` in versions of Python prior to 3.3, or
``platform.dist()`` on non-Linux systems), the corresponding marker
variable returned by setuptools will be set to the empty string.


Version numbers
---------------
.. list-table::
   :header-rows: 1

   * - Marker
     - Python equivalent
     - Sample values
   * - ``python_version``
     - ``platform.python_version()[:3]``
     - ``3.4``, ``2.7``
   * - ``python_full_version``
     - see definition below
     - ``3.4.0``, ``3.5.0b1``
   * - ``implementation_version``
     - see definition below
     - ``3.4.0``, ``3.5.0b1``

The ``python_full_version`` and ``implementation_version`` marker variables
are derived from ``sys.version_info`` and ``sys.implementation.version``
respectively, in accordance with the following algorithm::

    def format_full_version(info):
        version = '{0.major}.{0.minor}.{0.micro}'.format(info)
        kind = info.releaselevel
        if kind != 'final':
            version += kind[0] + str(info.serial)
        return version

    python_full_version = format_full_version(sys.version_info)
    implementation_version = format_full_version(sys.implementation.version)

``python_full_version`` will typically correspond to ``sys.version.split()[0]``.

If a particular version number value is not available (such as
``sys.implementation.version`` in versions of Python prior to 3.3) the
corresponding marker variable returned by setuptools will be set to ``0``


References
==========

.. [1] PEP 345, Metadata for Python Software Packages 1.2, Jones
   (http://www.python.org/dev/peps/pep-0345)

.. [2] PEP 0426, Metadata for Python Software Packages 2.0, Coghlan, Holth, Stufft
   (http://www.python.org/dev/peps/pep-0426)

.. [3] PEP 0440, Version Identification and Dependency Specification, Coghlan, Stufft
   (https://www.python.org/dev/peps/pep-0440/)

Copyright
=========

This document has been placed in the public domain.


..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
