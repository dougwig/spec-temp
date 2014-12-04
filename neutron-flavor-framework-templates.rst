..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Flavor framework - Templates and meta-data
==========================================

https://blueprints.launchpad.net/neutron/+spec/neutron-flavor-framework

The Flavor Framework spec introduces a static framework for creating different
feature levels for a service.  This proposal adds the ability to have the
objects being created for the service, influence the flavor's behavior.

Problem Description
===================

The original flavor framework allows operators to create custom feature levels
for a given service.  But, some features require per-instance data to be
effectively deployed.

An example would be a load balancer that offers page caching.  With flavors,
you can specify one flavor for no page caching, and another for page caching
everything.  But, full page caching often breaks legacy applications, and
certain URLs often need to be exempted, on a per load-balancer basis, such
as "do page caching, but not for /legacy/bank/thingie".

Another example would be an operator choosing to enable DoS protection for a
certain level of load balancer, but security features like those often need
to have a per-instance whitelist for the end-admin to be able to quickly deal
with false positives.

Proposed Change
===============

This proposal suggests two changes to the existing flavor framework:

* Add a model/table for "flavor object meta-data", which is meta-data that
  is stored for any given flavor-enabled neutron object, back-referenced with
  that objects' UUID, and a mixin (/me hides) that adds a "flavor_meta" attribute
  to said object.

* In the "metadata" field passed to drivers, support jinja templating syntax,
  with macro substitution available for the neutron model attributes or the
  above per-instance meta data.  Macro substitution will happen before the
  resulting "metadata" is passed to the backend plugin/driver, so drivers
  do not need to be aware of this templating.

Examples:

With the static flavor framework, you might have a flavor named "AwesomeSauce",
which includes load balancing and DDoS protection, and the driver metadata might
look like this:

::

  {
    vendor_z23_ddos_protection = true
  }

and that is exactly what would be passed to the z23 lbaas driver.  With the
templating syntax, that example becomes:

::

  {
    vendor_z23_ddos_protection = true
    z23_funky_whitelist = {{ meta.whitelist }}
  }

With a corresponsing flavor_meta field on the LoadBalancer object being:

::

  {
    whitelist = [ '1.2.3.4', '8.8.8.8' ]
  }

and the resulting metadata passed to the driver would be:

::

  {
    vendor_z23_ddos_protection = true
    z23_funky_whitelist = [ '1.2.3.4', '8.8.8.8' ]
  }

  
Data Model Impact
-----------------

One new logical models will be added to the database.

::

FlavorMeta
  id: uuid
  object_id: uuid
  meta: string

The following existing models will be modified with a flavor_meta attribute
pointing into the above table:

LoadBalancer

As more features are made flavor aware, their root models should add a flavor_meta
reference.

REST API Impact
---------------

Supporting flavor-enabled models will add an attribute for flavor_meta.

+------------+-------+---------+---------+------------+--------------+
|Attribute   |Type   |Access   |Default  |Validation/ |Description   |
|Name        |       |         |Value    |Conversion  |              |
+============+=======+=========+=========+============+==============+
|flavor_meta |string |RW, all  |''       |json string |per-object    |
|            |       |         |         |            |meta data     |
+------------+-------+---------+---------+------------+--------------+


Security Impact
---------------

None

Notifications Impact
--------------------

None

Other End User Impact
---------------------

This is an additional hook for operators, and will be invisible to end users.

Performance Impact
------------------

None

IPv6 Impact
-----------

None

Other Deployer Impact
---------------------

None

Developer Impact
----------------

Services/models that want to support flavors and this templating mechanism will
need to add the appropriate model entry and db migration.

Community Impact
----------------

This change allows operators greater flexibility in enabling advanced services
within the Neutron framework, without adding to community developer load for each
feature.

Alternatives
------------

* The first alternative is to do nothing.  This results in what many vendors are
  doing today, which is to brew up proprietary neutron solutions in order to
  expose more advanced features.  This results in inconsistent solutions for
  operators, more difficulty tracking trunk, and vendor lock-in.

* Another alternative is the same as this proposal minus the templating on the
  flavor metadata.  Since the flavor metadata is tied to a particular driver,
  and thus vendor specific, removing the templating would force vendors to expose
  vendor specific goo to their end users.  In addition, since multiple service
  profiles (drivers/vendors) can be used to implement a single flavor,
  not having templating would mean that that multiple backend support would break
  unless those backends supported the exact same back-end metadata, which
  is unlikely and impractical.

* Finally, the most straightforward alternative is to implement each feature
  natively into the services API.  In the aforementioned page caching example,
  add page caching as a feature in the API, with a URL exception list, and
  wait for all drivers to implement to that backend.  This adds maintenance
  and development load to the entire community, and means that Neutron will have
  a built-in lag for adding new features that appear in the marketplace.

Implementation
==============

Assignee(s)
-----------

https://launchpad.net/~dougwig

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.


Dependencies
============

* Main flavors framework
* LBaaS v2


Testing
=======

Please discuss how the change will be tested. We especially want to know what
tempest tests will be added. It is assumed that unit test coverage will be
added so that doesn't need to be mentioned explicitly, but discussion of why
you think unit tests are sufficient and we don't need to add more tempest
tests would need to be included.

Is this untestable in gate given current limitations (specific hardware /
software configurations available)? If so, are there mitigation plans (3rd
party testing, gate enhancements, etc).

Tempest Tests
-------------

Flavor tests need to be enhanced to include per-object meta-data and some basic
templated flavor metadata, and verify that substituted data is passed to
the backend.

Functional Tests
----------------

Tests to verify the flavor_meta field in models, and that the jinja substitution
is happening properly in the flavors code before being passed to backends.

API Tests
---------

Modify flavor API tests to include flavor_meta field for objects.


Documentation Impact
====================

User Documentation
------------------

This change is invisible to end users.

Developer Documentation
-----------------------

Deployers will need documentation for the new API fields and the templating syntax.

References
==========

* Flavors framework - https://review.openstack.org/#/c/102723
