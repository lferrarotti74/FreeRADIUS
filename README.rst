===================================
FreeRADIUS privacyIDEA Perl Plugin
===================================

This is the FreeRADIUS plugin to run with privacyIDEA. It lets a FreeRADIUS
server authenticate users against a `privacyIDEA <https://www.privacyidea.org>`_
server via its ``/validate/check`` REST endpoint, using the ``rlm_perl`` module.

.. contents:: Table of Contents
   :depth: 3
   :local:


Overview
========

The plugin is made of two parts that work together:

- ``privacyidea_radius.pm`` -- the Perl module loaded by FreeRADIUS's
  ``rlm_perl``. It builds the authentication request from the incoming
  RADIUS attributes, calls the privacyIDEA ``/validate/check`` endpoint,
  and maps the JSON response back into RADIUS reply attributes.
- ``rlm_perl.ini`` -- the configuration file read by the Perl module at
  startup. It defines the target URL, realm, resolver, SSL behaviour, and
  optional attribute-mapping rules.

FreeRADIUS itself is only told *when* to call the Perl module (via its own
``sites-enabled``/``mods-enabled`` configuration); the request details
(URL, realm, mapping rules, ...) all live in ``rlm_perl.ini``.


Repository Layout
==================

.. code-block:: text

    .
    |-- privacyidea_radius.pm        # The rlm_perl module (the actual logic)
    |-- rlm_perl.ini                 # Default/example configuration for the module
    |-- cpanfile                     # CPAN dependencies, used by CI
    |-- Changelog                    # Version history
    |-- LICENSE / copyright          # Licensing (GPLv2)
    |-- dictionary.netknights        # Custom RADIUS dictionary attributes
    `-- config/
        |-- freeradius2/
        |   |-- mods-perl-privacyidea    # rlm_perl module stanza (FreeRADIUS 2.x)
        |   `-- privacyidea               # Example virtual server (FreeRADIUS 2.x)
        `-- freeradius3/
            |-- mods-perl-privacyidea    # rlm_perl module stanza (FreeRADIUS 3.x)
            `-- privacyidea               # Example virtual server (FreeRADIUS 3.x)

The files under ``config/`` are **reference templates**. In a real
deployment they are copied/adapted into FreeRADIUS's own configuration
tree, e.g. ``/etc/freeradius/3.0/mods-enabled/`` and
``/etc/freeradius/3.0/sites-enabled/``.


Requirements
============

Perl dependencies (see ``cpanfile``):

- ``LWP`` (version 6, for SSL options support)
- ``Config::IniFiles``
- ``Data::Dump``
- ``Try::Tiny``
- ``JSON``
- ``Time::HiRes``
- ``URI::Encode``
- ``Encode::Guess`` (Perl core module, no separate install needed)

FreeRADIUS with ``rlm_perl`` support (2.x or 3.x).


Installation
============

FreeRADIUS 3.x
--------------

1. Copy ``privacyidea_radius.pm`` to a path readable by FreeRADIUS, e.g.
   ``/usr/share/privacyidea/freeradius/privacyidea_radius.pm``.
2. Install the module stanza, based on ``config/freeradius3/mods-perl-privacyidea``::

       perl perl-privacyidea {
           filename = /usr/share/privacyidea/freeradius/privacyidea_radius.pm
       }

   into ``/etc/freeradius/3.0/mods-enabled/`` (symlinked from ``mods-available/``
   as usual).
3. Install a virtual server, based on ``config/freeradius3/privacyidea``, into
   ``/etc/freeradius/3.0/sites-enabled/``.
4. Place ``rlm_perl.ini`` in one of the locations the module searches (see
   `Configuration File Lookup`_ below) and adjust it for your environment.
5. Restart/reload FreeRADIUS and test with ``radiusd -X``.

FreeRADIUS 2.x
--------------

Same idea, using ``config/freeradius2/mods-perl-privacyidea`` (uses
``module =`` instead of ``filename =``) and ``config/freeradius2/privacyidea``
as templates. The 2.x example additionally chains ``preprocess``, ``digest``,
``suffix``, ``ntdomain``, ``expiration``, ``logintime`` and ``pap`` in
``authorize``, reflecting the broader default FreeRADIUS 2.x site.


Configuration
=============

Configuration File Lookup
--------------------------

At module load time, ``privacyidea_radius.pm`` searches for its ini file in
this fixed order (first match wins)::

    /etc/privacyidea/rlm_perl.ini
    /etc/freeradius/rlm_perl.ini
    /opt/privacyIDEA/rlm_perl.ini

This can be overridden per module instance, which allows **several
independent instances of the same script** to use **different configuration
files** -- e.g. to talk to different privacyIDEA servers from the same
FreeRADIUS server::

    perl privacyIDEA-A {
        filename = /usr/share/privacyidea/freeradius/privacyidea_radius.pm
        config {
            configfile = /etc/privacyidea/rlm_perl-A.ini
        }
    }

This multi-instance/multi-config capability was introduced in version 3.4
(see ``Changelog``, *"Allow different configs with the same script to use
redundant to ask different privacyIDEA servers"*) and is the foundation for
the scoped-configuration approach described in `Extending: Multiple Backends
/ Scoped Configuration`_.

The ``[Default]`` Section
---------------------------

``rlm_perl.ini`` always has one mandatory section, ``[Default]``, read once
at module load::

    [Default]
    URL = https://localhost/validate/check
    #REALM = someRealm
    #RESCONF = someResolver
    SSL_CHECK = false
    #SSL_CA_PATH =
    #DEBUG = true

================  =================================================================
Key               Meaning
================  =================================================================
``URL``           privacyIDEA ``/validate/check`` endpoint to call.
``REALM``         Default privacyIDEA realm, if not taken from the RADIUS request.
``RESCONF``       Resolver configuration name to pass along.
``SSL_CHECK``     ``true``/``false`` -- whether to verify the server's SSL certificate.
``SSL_CA_PATH``   CA bundle path used when ``SSL_CHECK`` is ``true``.
``DEBUG``         ``true``/``false`` -- verbose logging of request parameters.
``TIMEOUT``       HTTP request timeout in seconds (default: ``10``).
``CLIENTATTRIBUTE``  RADIUS attribute to use as the ``client`` IP sent to privacyIDEA.
``SPLIT_NULL_BYTE``  ``true``/``false`` -- split the password at a NUL byte.
``ADD_EMPTY_PASS``   ``true``/``false`` -- send an empty password if none was supplied.
================  =================================================================

Any of these keys may **also** be placed in an auth-type-specific section
(see `How a Request Is Processed`_) to override the default per request.

Attribute Mapping Sections
----------------------------

``[Mapping ...]``
~~~~~~~~~~~~~~~~~~

Copies a field from the privacyIDEA JSON response into a RADIUS reply
attribute::

    [Mapping user]
    # detail -> user -> group  becomes the RADIUS "Class" attribute
    group = Class

``[Attribute ...]``
~~~~~~~~~~~~~~~~~~~~

Performs regex-based attribute mangling against (possibly multi-valued)
privacyIDEA user attributes, optionally with a fixed radiusAttribute name,
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

Section names after ``Attribute``/``Mapping`` (e.g. ``Filter-Id``,
``otherAttribute``, ``user``) are arbitrary labels -- you can define as many
``[Attribute ...]`` sections as needed, one per RADIUS attribute rule.


How a Request Is Processed
===========================

1. FreeRADIUS's ``authorize {}`` section calls the ``perl-privacyidea``
   module instance. The module's own ``authorize()`` subroutine always
   returns ``RLM_MODULE_OK`` -- it does no real work at this stage.
2. The site config then forces a fixed ``Auth-Type := Perl`` (see
   ``config/freeradius3/privacyidea``), which routes the request to the
   matching block in ``authenticate {}``.
3. Inside the module's ``authenticate()``:

   - ``$Config->{URL}`` (loaded from ``[Default]``) is the starting point.
   - The module reads ``$RAD_CONFIG{"Auth-Type"}`` and, **if an ini section
     with that exact name exists**, any key present there (``URL``,
     ``REALM``, ``RESCONF``, ``SSL_CHECK``, ``SSL_CA_PATH``, ``TIMEOUT``,
     ``CLIENTATTRIBUTE``, ``DEBUG``, ``SPLIT_NULL_BYTE``,
     ``ADD_EMPTY_PASS``) overrides the default.
   - With the current shipped site config, ``Auth-Type`` is always
     ``"Perl"``, and since no ``[Perl]`` section exists in ``rlm_perl.ini``,
     this override path is effectively **dormant** -- every request uses
     ``[Default]``.

4. The module builds the HTTP POST body (``user``, ``pass``, ``realm``,
   ``resConf``, ``client``, ``state``), URL-encodes user/password, and
   calls privacyIDEA's ``/validate/check``.
5. The JSON response is parsed:

   - ``result.value == true`` -> access granted (``RLM_MODULE_OK``).
   - ``result.status == true`` with a ``transaction_id`` -> challenge/response
     (``RLM_MODULE_HANDLED``, ``Access-Challenge``).
   - ``result.status == true`` without a ``transaction_id`` -> access denied
     (``RLM_MODULE_REJECT``).
   - ``result.status == false`` -> backend/internal error
     (``RLM_MODULE_FAIL``, or ``RLM_MODULE_NOTFOUND`` for error code 904).

6. Any ``[Mapping ...]``/``[Attribute ...]`` rules are applied to enrich the
   RADIUS reply with additional attributes.


Extending: Multiple Backends / Scoped Configuration
=====================================================

This chapter documents a **not-yet-implemented, forward-looking** extension:
using ``Auth-Type`` to route different requests to different privacyIDEA
backends/realms, each defined as its own section in ``rlm_perl.ini``. No
code or configuration change described here is currently applied -- this is
a design guide for when/if it becomes necessary.

Concept
-------

The ``Auth-Type``-to-ini-section lookup already present in
``privacyidea_radius.pm`` (see step 3 above) is a **generic mechanism** that
is simply unused today, because the shipped site config always sets
``Auth-Type := Perl``. To activate it, two independent conditions must both
be true for a given request:

1. ``$RAD_CONFIG{"Auth-Type"}`` must be set to something other than the
   fixed ``"Perl"`` value, **before** ``authenticate {}`` runs.
2. An ini section with that exact name must exist, containing the
   overrides you want (typically ``URL`` and/or ``REALM``).

Why No Perl Code Changes Are Needed
---------------------------------------

Because the lookup is already generic (keyed by whatever string
``Auth-Type`` holds), adding a new scope is **purely a configuration
change**:

- a new routing rule in the FreeRADIUS site file, and
- a new section in ``rlm_perl.ini``.

``privacyidea_radius.pm`` itself never needs to be touched to add, remove,
or change a scope. This matters when the team that manages the FreeRADIUS
site configuration is not the same team that manages the privacyIDEA
Perl/ini configuration (or vice versa): each side only ever edits its own
file type.

Step 1 -- FreeRADIUS Site Configuration
------------------------------------------

Add a routing condition in ``authorize {}`` *before* the existing
unconditional ``Auth-Type := Perl`` fallback, and guard that fallback so it
does not clobber a scope that was just set. Then add a matching
``Auth-Type <scope> { perl-privacyidea }`` block in ``authenticate {}``::

    authorize {
        update request {
            Packet-Src-IP-Address = "%{Packet-Src-IP-Address}"
        }

        # --- scope selection (example criterion: NAS IP) ---
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

The criterion used for scope selection is deployment-specific -- NAS IP
address, realm suffix on the username, ``Calling-Station-Id``, a
Huntgroup, etc. -- and is the main design decision to make before adopting
this pattern.

Step 2 -- ``rlm_perl.ini`` Sections
--------------------------------------

Add one section per scope, with only the keys that should differ from
``[Default]``::

    [Default]
    URL = https://localhost/validate/check
    SSL_CHECK = false

    [scope1]
    URL = https://privacyidea-node-a.example.com/validate/check
    REALM = realmA

    [scope2]
    URL = https://privacyidea-node-b.example.com/validate/check
    REALM = realmB

Worked Example
-----------------

A request arrives from NAS ``192.168.10.5``:

1. ``authorize {}`` sets ``control:Auth-Type = scope1``.
2. The guarded fallback leaves it untouched (it is already set).
3. FreeRADIUS dispatches to ``authenticate { Auth-Type scope1 { ... } }``.
4. The module finds ``$RAD_CONFIG{"Auth-Type"} eq "scope1"``, looks up the
   ``[scope1]`` section, and overrides ``URL``/``REALM`` accordingly before
   calling privacyIDEA.

Design Considerations
------------------------

- **Site reload required.** Changes to the FreeRADIUS site file need a
  config test (``radiusd -X``) and a service reload/restart to take
  effect; this is less "hot" than an ini-only change.
- **``[Default]`` stays the safe fallback.** Any request whose criterion
  does not match a defined scope keeps using ``[Default]`` and ``Auth-Type
  = Perl`` -- no behaviour change for existing traffic.
- **Keep the guard.** Without ``if (!control:Auth-Type) { ... }``, the
  existing unconditional assignment at the end of ``authorize {}`` would
  silently overwrite any scope you set earlier, and every request would
  fall back to ``[Default]`` regardless of the new routing rules.
- **One ``authenticate`` block per scope is mandatory.** FreeRADIUS
  dispatches strictly on the ``Auth-Type`` value; a scope with no matching
  block in ``authenticate {}`` will fail even though the ini section and
  routing condition are correct.

Related Idea: Per-Scope Failover URLs
-----------------------------------------

Orthogonal to scope routing, the ``URL`` key inside any section (including
``[Default]``) could in principle hold a comma-separated list of
privacyIDEA endpoints, tried in order until one responds successfully, to
provide high-availability failover without a full scope setup. This would
require a code change in ``privacyidea_radius.pm`` (looping over
candidate URLs in ``authenticate()``) and is **not implemented** -- it is
noted here only as a related, independently useful extension to consider
alongside scoped configuration.


Debugging and Logging
=======================

- Set ``DEBUG = true`` in ``[Default]`` (or an auth-type section) to log
  full request parameters and the raw privacyIDEA JSON response.
- All logging goes through ``radiusd::radlog`` at various levels (``Debug``,
  ``Info``, ``Error``, ...) and shows up in FreeRADIUS's own log output,
  or on the console when running ``radiusd -X``.
- ``SSL_CHECK = false`` disables certificate verification entirely --
  intended for lab/test setups only; use ``SSL_CHECK = true`` with
  ``SSL_CA_PATH`` (or the system CA store) in production.


Changelog and License
=======================

See ``Changelog`` for the full version history and ``LICENSE``/``copyright``
for licensing terms (GPLv2).
