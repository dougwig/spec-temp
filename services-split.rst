..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================================
Proposal for Neutron Services repo split
========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/neutron/+spec/services-split

This spec outlines the technical changes required for the repo split.

The following intro is shamelessly stolen from an email thread started by Mark McClain:

Over the last several months, the members of the Networking Program have been
discussing ways to improve the management of our program.  When the Quantum
project was initially launched, we envisioned a combined service that included
all things network related.  This vision served us well in the early days as
the team mostly focused on building out layers 2 and 3; however, we’ve run into
growth challenges as the project started building out layers 4 through 7.
Initially, we thought that development would float across all layers of the
networking stack, but the reality is that the development concentrates around
either layer 2 and 3 or layers 4 through 7.  In the last few cycles, we’ve also 
discovered that these concentrations have different velocities and a single
core team forces one to match the other to the detriment of the one forced to
slow down.

Going forward we want to divide the Neutron repository into multiple separate
repositories lead by a common Networking PTL.  The current mission of the
program will remain unchanged.


Problem Description
===================

This proposal deals with the technical aspects of splitting the neutron repo
into two repos, one for basic L2/L3 plumbing, one for advanced services.  The
repos will be referred to as "neutron" and "services" within this spec, until
a project name for the "services" repo is selected.

* Currently the neutron code is in a single repo, with a single set of cores.

* After the split, all "services" code and db models will be in a new
  services repo, preserving history.

* After the split, no services code to be in the neutron repo, with
  non-relavant history pruned.

* During the split, an infra/project-config change for the new repo, shared
  spec repo, and new core team will be created.

* Existing services code must be supported until deprecation.

* Existing services fixes must be backported into neutron/stable.

* The split needs to be aware of the imminent REST refactor.


Proposed Change
===============

* The repo will be split via infra scripts which will preserve history for all
  changes.

* The existing services code (e.g. lbaas v1), will move into the services repo
  and be maintained by that repo's team going forward.

* Initially, the services repo will not have its own REST service, and will
  utilize neutron extensions to share the neutron API namespace.  Initially,
  service plugins will be moved (Kilo-1); after the neutron REST refactor,
  service extensions will be moved (Kilo-3), and the issue of a separate
  service endpoint will be revisited in the L timeframe or later.  This is to
  maintain API consistency during the split process while the new repositories
  are being worked out.

* The services repo will include neutron as a dependency in the
  requirements.txt file, and the services code may import neutron as a library.

* The neutron repo will have some manner of dependency specified for packaging,
  so that installs of neutron will pull in the services repo in the short-term.
  A hacking check will be added to ensure that neutron code does not import
  services code as a library.

* Extensions will stay inside the neutron repo, at least until after the REST
  refactor.

* Code merged onto the existing 'feature/lbaasv2' neutron feature branch will
  be merged into the service repo as part of the split.

* All outstanding gerrit reviews for services, currently submitted against 
  Neutron, will have to be abandoned and resubmitted against the services repo.

* Tox will pass cleanly in both projects immediately post-split.

* The services repo will not support python 2.6.

* Backported fixes will be merged into neutron/stable branches by the
  "services" team, approved by the stable team.

* The services repo will use its own database (see Data Model Impact)

* The services repo will have its own config file.

Data Model Impact
-----------------

Services data models will be moved to the service repo, and removed from
neutron.

The services repo will have its own database migration chain.

An initial db migration state will be created by starting from the neutron db state as of the split and stripping non-service related tables.

A db migration will be added to neutron to strip service tables.

An upgrade script will be provided to migrate db data from neutron to services.


REST API Impact
---------------

The REST API will be identical before and after the split.

Security Impact
---------------

None.

Notifications Impact
--------------------

None.

Other End User Impact
---------------------

None.

Performance Impact
------------------

None.

Other Deployer Impact
---------------------

The new services project will have its own database and config file.  In
addition, by the end of Kilo, neutron will need to load the services API
extension.

For Kilo, neutron will assume that the services repo exists, and include the
path to its API extensions in the conf file by default.

A db/conf upgrade command-line script will be provided, which will copy
relevant database tables and configuration elements to the new db and INI file.

In Kilo, a fresh install will end up doing the following steps:

* Install neutron.  Services package will be pulled in as a dependency,
its installer will run before neutron, initializing db, writing default config,
then neutron will install as normal.

* Deployer will edit neutron.conf for db and other info.

* Deployer will edit services-tron.conf for db and other info.

* Deployer will need to restart neutron-server.

In Kilo, an upgrade from Juno or Icehouse will do the following steps:

* Install neutron.  Services package will be pulled in as a dependency,
its installer will run before neutron, initializing db, writing default config,
then neutron will install as normal.

* Deployer will edit neutron.conf for db and other info.

* Deployer will edit services-tron.conf for db and other info.

* Deployer will run services-db-migration script.

* Deployer will need to restart neutron-server.

In the upgrade scenario, the REST controller will bounce, but active services
(load balancers, etc) will remain active.

Open issue: between the install and the server restart, neutron-server will
return errors for service API operations.  How to alleviate that, or is that
an orchestration issue?


Developer Impact
----------------

Anyone importing neutron.services will have to import the new project modules instead.

Community Impact
----------------

This split was discussed at the Neutron summit, the openstack-dev mailing
list, and multiple IRC meetings.

Alternatives
------------

* Do nothing and keep it all in one repo.

* Services to stackforge.

* Services split with its own REST server endpoint.

* Services shares neutron db and config.

* Modify gerrit to allow different core teams in one repo.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  https://launchpad.net/~dougwig

Other contributors:
  https://launchpad.net/~mestery

Work Items
----------

* Identify files for each repo.

* Adapt olso graduation script for git split.

* Merge in feature branch.

* Adjust imports in new repo.

* Add requirements to each project.

* Add hacking rule to neutron.

* Verify or add neutron's ability to load out-of-tree service plugins.

* Create initial services db migration files.

* Neutron db migration to strip services data (to be applied later!)

* Fix references to neutron in various files (e.g. README)

* Finalize project name

* Infra patch to create new repo/group

* Get unit tests passing cleanly.

* Upgrade script to migrate db and config data.


Dependencies
============

* Infra creating separate repos.

* REST refactor not colliding at the same time.  This needs to happen before
  or after.


Testing
=======

* Unit tests will split between repos, matching the code split.

* Tempest tests will initiall remain unchanged, as the service endpoint will
  be identical before and after the split.  Setup steps that touch db and/or
  config files may need to be updated to reflect new locations.

Tempest Tests
-------------

Unchanged.

Functional Tests
----------------

Unchanged.

API Tests
---------

Unchanged.


Documentation Impact
====================

Advanced services documentation should be separated from the Neutron
documentation.

User Documentation
------------------

Documentation referencing neutron.conf and the neutron db will need to be
modified to reflect the new config file and database.

Developer Documentation
-----------------------

Documentation referencing neutron.conf and the neutron db will need to be
modified to reflect the new config file and database.


Q & A
=====

* Split or shared CLI/client?

** Answer: Shared until REST service endpoing split.

* Do we take this opportunity to re-org directories?


References
==========

* https://etherpad.openstack.org/p/neutron-services

* http://lists.openstack.org/pipermail/openstack-dev/2014-November/050961.html

* Other spec?
