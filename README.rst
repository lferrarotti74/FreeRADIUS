===================================
FreeRADIUS privacyIDEA Perl Plugin
===================================

This is the FreeRADIUS plugin to run with privacyIDEA. It lets a FreeRADIUS
server authenticate users against a `privacyIDEA <https://www.privacyidea.org>`_
server via its ``/validate/check`` endpoint, using the ``rlm_perl`` module.

.. contents:: Table of Contents
   :depth: 2
   :local:


Overview
========

The plugin consists of two parts:

- ``privacyidea_radius.pm`` -- the Perl module loaded by FreeRADIUS's
  ``rlm_perl``. It builds the request from the incoming RADIUS attributes,
  calls privacyIDEA and maps the response back to RADIUS reply attributes.
- ``rlm_perl.ini`` -- the configuration file read by the module. It defines
  the privacyIDEA URL, realm, SSL behaviour and optional attribute mappings.

FreeRADIUS only decides *when* to call the module; the details of the request
are configured in ``rlm_perl.ini``.


Repository Layout
=================

.. code-block:: text

    .
    |-- privacyidea_radius.pm        # The rlm_perl module
    |-- rlm_perl.ini                 # Example configuration for the module
    |-- cpanfile                     # CPAN dependencies
    |-- Changelog                    # Version history
    |-- dictionary.netknights        # Custom RADIUS dictionary attributes
    `-- config/
        |-- freeradius2/             # Example module and virtual server (2.x)
        `-- freeradius3/             # Example module and virtual server (3.x)

The files under ``config/`` are templates. Copy and adapt them into the
FreeRADIUS configuration tree, e.g. ``/etc/freeradius/3.0/mods-enabled/`` and
``/etc/freeradius/3.0/sites-enabled/``.


Requirements
============

FreeRADIUS with ``rlm_perl`` support (2.x or 3.x) and the Perl modules listed
in ``cpanfile``:

- ``LWP`` (version 6, for SSL options)
- ``Config::IniFiles``
- ``Data::Dump``
- ``Try::Tiny``
- ``JSON``
- ``Time::HiRes``
- ``URI::Encode``


Installation
============

1. Copy ``privacyidea_radius.pm`` to a path readable by FreeRADIUS, e.g.
   ``/usr/share/privacyidea/freeradius/privacyidea_radius.pm``.
2. Add the module configuration, based on
   ``config/freeradius3/mods-perl-privacyidea``::

       perl perl-privacyidea {
           filename = /usr/share/privacyidea/freeradius/privacyidea_radius.pm
       }

3. Add a virtual server, based on ``config/freeradius3/privacyidea``.
4. Place ``rlm_perl.ini`` where the module can find it (see
   `Configuration`_) and adjust it.
5. Restart FreeRADIUS. Use ``radiusd -X`` to test.

For FreeRADIUS 2.x use the templates in ``config/freeradius2/`` instead.


Configuration
=============

The module looks for its configuration file in this order and uses the first
one found::

    /etc/privacyidea/rlm_perl.ini
    /etc/freeradius/rlm_perl.ini
    /opt/privacyIDEA/rlm_perl.ini

A different file can be set per module instance with ``configfile``::

    perl perl-privacyidea {
        filename = /usr/share/privacyidea/freeradius/privacyidea_radius.pm
        config {
            configfile = /etc/privacyidea/rlm_perl.ini
        }
    }

The ``[Default]`` Section
-------------------------

::

    [Default]
    URL = https://localhost/validate/check
    #REALM = someRealm
    #RESCONF = someResolver
    SSL_CHECK = false
    #SSL_CA_PATH =
    #DEBUG = true

===================  ==========================================================
Key                  Meaning
===================  ==========================================================
``URL``              The privacyIDEA ``/validate/check`` endpoint.
``REALM``            Realm sent to privacyIDEA.
``RESCONF``          Resolver configuration name sent to privacyIDEA.
``SSL_CHECK``        ``true``/``false``: verify the server certificate.
``SSL_CA_PATH``      CA path used to verify the server certificate.
``TIMEOUT``          HTTP request timeout in seconds (default: ``10``).
``CLIENTATTRIBUTE``  RADIUS attribute used as the ``client`` IP.
``SPLIT_NULL_BYTE``  ``true``/``false``: split the password at a NUL byte.
``ADD_EMPTY_PASS``   ``true``/``false``: send an empty password if none is set.
``DEBUG``            ``true``/``false``: verbose logging.
===================  ==========================================================

Mapping privacyIDEA Responses to RADIUS Attributes
--------------------------------------------------

``[Mapping ...]`` copies a value of the privacyIDEA response into a RADIUS
reply attribute::

    [Mapping user]
    # detail -> user -> group is returned as the RADIUS attribute "Class"
    group = Class

``[Attribute ...]`` matches (multi-valued) user attributes with a regular
expression and adds the result as a RADIUS attribute, optionally with a
prefix and suffix::

    [Attribute Filter-Id]
    dir = user
    userAttribute = acl
    regex = CN=(\w*)-user,OU=sales,DC=example,DC=com

    [Attribute otherAttribute]
    radiusAttribute = Filter-Id
    userAttribute = user-resolver
    regex = resolver1
    prefix = FIXEDValue

The text after ``Attribute`` or ``Mapping`` is only a label, so several
sections can be defined. See the comments in ``rlm_perl.ini`` for details.


Using Different Configurations per Auth-Type
============================================

With a single ``[Default]`` section every request is sent to the same
privacyIDEA server and realm. If requests should be handled differently,
e.g. sent to different privacyIDEA servers or realms, the module can read
additional ini sections named after the FreeRADIUS ``Auth-Type``.

If a section with the name of the current ``Auth-Type`` exists, the keys
listed in `The [Default] Section`_ that are set in it replace the values from
``[Default]`` for that request. All other keys keep their default value.

Example ``rlm_perl.ini``::

    [Default]
    URL = https://localhost/validate/check
    SSL_CHECK = false

    [scope1]
    URL = https://privacyidea-a.example.com/validate/check
    REALM = realmA

    [scope2]
    URL = https://privacyidea-b.example.com/validate/check
    REALM = realmB

Set the ``Auth-Type`` in the ``authorize`` section of the virtual server and
add an ``Auth-Type`` block for each name in ``authenticate``. This example
selects the section by the NAS IP address; any other attribute of the request
can be used::

    authorize {
        if (NAS-IP-Address == 192.168.10.5) {
            update control {
                Auth-Type := scope1
            }
        }
        elsif (NAS-IP-Address == 192.168.10.6) {
            update control {
                Auth-Type := scope2
            }
        }

        perl-privacyidea

        if (ok || updated) {
            if (!control:Auth-Type) {
                update control {
                    Auth-Type := Perl
                }
            }
        }
    }

    authenticate {
        Auth-Type Perl {
            perl-privacyidea
        }
        Auth-Type scope1 {
            perl-privacyidea
        }
        Auth-Type scope2 {
            perl-privacyidea
        }
    }

Notes:

- The section name in ``rlm_perl.ini`` must match the ``Auth-Type`` exactly.
- Requests that do not match a condition use ``Auth-Type Perl`` and the
  ``[Default]`` section, as in the shipped example configuration.
- Changes to the virtual server need a FreeRADIUS reload; changes to
  ``rlm_perl.ini`` are read when the module is loaded.
- To use several independent ini files instead, define several ``perl``
  module instances with different ``configfile`` values (see
  `Configuration`_).


Debugging
=========

- Set ``DEBUG = true`` (in ``[Default]`` or in an ``Auth-Type`` section) for
  verbose logging.
- Log output goes to the FreeRADIUS log, or to the console with
  ``radiusd -X``.
- ``SSL_CHECK = false`` disables certificate verification and is meant for
  test setups. Use ``SSL_CHECK = true`` with ``SSL_CA_PATH`` in production.


Changelog and License
=====================

See ``Changelog`` for the version history and ``LICENSE`` / ``copyright`` for
the license terms.
