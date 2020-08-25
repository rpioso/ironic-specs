..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================================
Efficient and accelerated single-step system hardware configuration
===================================================================

https://storyboard.openstack.org/#!/story/2003594

Ironic's popularity and the number of ironic deployments, both integrated
OpenStack and stand-alone, continue to grow. Alongside that, the number of bare
metal systems managed by each deployment is increasing. Those trends have led
operators to demand that improvements be made to the operational management of
hardware system configuration -- BIOS settings, RAID, boot mode, network ports,
inventory, and more. Solutions must reduce operational costs and the time
required to fulfill a tenant's request for a newly deployed and activated
bare metal system.

Use cases which would benefit from this proposal include:

* Correct, consistent, and rapid hardware configuration during deployment of
  fleets of bare metal systems with similar hardware capabilities
* Production environments in which physical systems are repurposed for
  workloads that require different configurations and fast turnaround is at a
  premium
* Inventory of individual system configurations

To achieve that, prospective solutions should offer:

* Creation of bare metal system configurations from common or "golden" systems
* Management of configurations stored at a designated storage location
* Application of a stored configuration to a bare metal system
* Acquisiton of a system's resulting entire configuration following application
  of a complete or partial configuration
* Acceleration of hardware configuration by reducing the amount of time, often
  via fewer required reboots, which is dependent on system and ironic hardware
  type support

This specification proposes a generally applicable solution comprised of new
cleaning and deploy steps which process system configurations. Each hardware
type may implement it according to its and its system's capabilities.


Problem description
===================

A large number of systems with identical or similar hardware arrives. The user
wants all of their hardware configured the same.

* In ironic, that can be a tedious, error-prone task.
* Ironic is not capable of configuring all of the hardware subsystems for which
  vendors offer configurability.
* Applying a configuration can take a great deal of time, often requiring
  multiple reboots.

The same challenges apply when different workflows are run on the same system,
each needing a different hardware configuration.

Proposed change
===============

Introduction
------------

The following concept is used in this specification:

**configuration mold**
    A document which expresses the current or desired configuraton of a
    hardware system. It also offers an inventory of a system's hardware
    configuration.

This specification proposes to:

* Add new, optional cleaning [1]_ and deploy [2]_ steps which, in a single
  step, can completely configure a system's hardware. They can apply the saved
  configuration of a system with sufficiently similar hardware capabilities.
  Steps which obtain and persistently store a system's configuration are also
  proposed. They will be defined by the driver management hardware interface,
  ``class ManagementInterface`` in ``.../ironic/drivers/base.py``.

  * ``export_configuration`` --
        obtain a system's hardware configuration and persistently store it as a
        *configuration mold* so it can be used to configure other, sufficiently
        capable systems and as an inventory. More details are available in
        `Export configuration`_.
  * ``import_configuration`` --
        apply the hardware configuration described by an existing, persistently
        stored *configuration mold* to a sufficently capable system. More
        details are available in `Import configuration`_.
  * ``import_export_configuration`` --
        import and then export the hardware configuration of the same system as
        a single, atomic step from the perspective of ironic's cleaning and
        deployment mechanisms. This can further reduce the time required to
        provision a system than a sequence of separate import and export steps.
        More details are available in `Import and export configuration`_.

* Define the data format of the vendor-independent content of a *configuraton
  mold*. It contains a section for each hardware subsystem it supports, RAID and
  BIOS. The desired configuration of a subsystem for which ironic has been
  offering configuration support, again RAID and BIOS, is expressed as much as
  possible in ironic terms.

  To support hardware configuration beyond what ironic defines, but which
  vendors, systems, and their hardware types could offer in a system-specific
  manner, a third section, OEM, is proposed. The OEM section can also be used
  to specify configuration ironic already supports. The content of the OEM
  section is specific to the system and its hardware type.

  More details, along with an example, are in section
  `Format of configuration data`_.

* Describe the persistent storage of *configuration molds* for both integrated
  OpenStack and stand-alone ironic environments and how ironic and its users
  interact with them. See `Storage of configuration data`_.

* Declare what is considered a complete implementation for the purposes of this
  specification. An implementation will be  considered complete if it offers
  all three (3) pairs of cleaning and deploy steps and supports all of the
  defined *configuration mold* sections, presently BIOS and RAID. Additionally,
  persistent storage of *configuration molds* must be supported for both ironic
  environments, integrated OpenStack and stand-alone.

* Offer an alternative, not a replacement, to ironic's currently available
  granular configuration steps which operate on a single subsystem: RAID and
  BIOS. How it compares to ironic's present functionality is described in
  `Alternatives`_.

* Outline a minimum viable product (MVP) implementation by the ``idrac``
  hardware type in section `idrac-redfish implementation`_. We aim for it to
  offer all three (3) pairs of cleaning and deploy steps. In the MVP, only the
  OEM *configuration mold* section will be supported. Both methods of
  persistent *configuration mold* storage will be implemented and supported.

  .. note ::
      However, ironic deployment mechanism limitations may restrict them to be
      only cleaning steps until those can be addressed.

Goals
-----

With this specification, we are going to achieve the following goals:

* Define an ironic hardware type framework which offers more operationally
  efficient bare metal hardware configuration
* Facilitate a consistent method to accelerate bare metal hardware
  configuration, necessitating fewer reboots, when supported by a hardware
  system and its ironic hardware type
* Describe a first, MVP implementation by the ``idrac`` hardware type

Non-goals
---------

The following are considered outside the scope of this specification:

* Implementation of the approach by all hardware types; it is optional
* Requiring a hardware type’s implementation be complete; it may be partial

Export configuration
~~~~~~~~~~~~~~~~~~~~

The export configuration clean/deploy step extracts existing configuration of
indicated server ("golden server") and stores it in designated storage location
to be used in `Import configuration`_ clean/deploy step.

Clean/deploy step details are:

Interface
  Management interface
Name
  ``export_configuration``
Details
  Gets the configuration of the server against which the step is run and
  stores it in specific format in indicated storage as configured by Ironic.
Priority
  0
Stoppable
  No
Arguments
  * Name of storage item to save the configuration to


Sample of clean/deploy step configuration:

.. code-block::

  {
    "interface": "management",
    "step": "export_configuration",
    "args": {
      "configuration_name": "edge_dell_emc-poweredge_r640.json"
    }
  }

The workflow of configuration export consists of 3 parts:

1. Get current node's configuration (driver specific)
2. Transform the configuration to common format
   (common to all drivers, see `Format of configuration data`_)
3. Save the storage item to designated storage (common to all drivers,
   see `Storage of configuration data`_)


Usage of `export_configuration` is not mandatory. If the configuration is
acquired previously or in another way, user can also upload the configuration
directly to the storage without using Ironic - when having Swift for storage,
then using Swift API or CLI, or when having web server, then writing file to
filesystem of web server's folder.


Import configuration
~~~~~~~~~~~~~~~~~~~~

Once the configuration is available, user can use it in the import
configuration clean/deploy step to configure the servers.

Clean/deploy step details are:

Interface
  Management interface
Name
  ``import_configuration``
Details
  Gets pre-created configuration from storage by given item name and imports
  that into given server.
Priority
  0
Stoppable
  No
Arguments
  * Name of storage item to use to fetch desired configuration


Sample:

.. code-block::

  {
    "interface": "management",
    "step": "import_configuration",
    "args": {
      "configuration_name": "edge_dell_emc-poweredge_r640.json"
    }
  }

The workflow of the import configuration consists of 3 parts:

1. Using given configuration name and Ironic's storage settings, get the
   configuration from the storage (common to all drivers)
2. Transform the configuration to driver specific format (driver specific)
3. Apply the configuration (driver specific) and prospectively update the
   node's cached BIOS configuration and ``raid_config`` field as it is already
   done in corresponding BIOS and RAID clean and deploy steps.

Sections that are not specified in the configuration mold are left intact, for
example, it is possible to configure only subset of BIOS settings and other
BIOS settings and RAID settings remain unchanged.

If an error is encountered, the clean/deploy step fails. On failure, no
assurances can be made about the state of the system's configuration nor side
effects on the ironic node fields, because the application of the configuration
is system and ironic hardware type dependent and there are many possible
failure modes. A defined subsystem configuration sequence and transactional
rollback semantics do not seem to apply.

When a step fails, the ironic node is placed in the ``clean failed`` or
``deploy failed`` state and the node's ``last_error`` field may contain further
information about the cause of the failure.

Further improvement could be additional processing on the configuration data
before applying, for example, clean up irrelevant parts, but this is currently
not in the scope of this RFE.

.. warning ::
  Depending on each vendor's capabilities importing can be powerful step that
  allows configuring various things. Users and vendors need to be aware of
  these capabilities and make sure not to overwrite settings that are not
  intended to be replaced, for example, deleting RAID settings or static BMC
  IP address.


Import and export configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Import and export configuration clean/deploy step is composite step that
executes both importing and exporting one after another as atomic operation.
This can be used to get the inventory just after configuration and can be
useful when not all aspects of system are being configured, but need to know
the outcome for all aspects.

Clean/deploy step details are:

Interface
  Management interface
Name
  ``import_export_configuration``
Details
  Gets pre-created configuration from storage, imports that into given server
  and exports resulting configuration.
Priority
  0
Stoppable
  No
Arguments
  * For ``import`` section:

    * Name of storage item to use to fetch desired configuration for import
  * For ``export`` section:

    * Name of storage item to save the configuration to
    * For ``raid`` settings:

      * Optional flag ``enable_create_nonroot_volumes`` to set
        ``create_nonroot_volumes`` to ``true`` in exported data, defaults
        to ``false``
      * Optional flag to ``enable_create_root_volume`` to set
        ``create_root_volume`` to ``true`` in exported data, defaults to
        ``false``
      * Optional flag ``enable_delete_existing`` to set ``delete_existing``
        to ``true`` in exported data, defaults to ``false``


Sample of clean/deploy step configuration:

.. code-block::

  {
    "interface": "management",
    "step": "import_export_configuration",
    "args": {
      "import": {
        "configuration_name": "edge_dell_emc-poweredge_r640.json"
      },
      "export": {
        "configuration_name": "edge_dell_emc-poweredge_r640_server005.json",
        "raid": {
          "enable_create_nonroot_volumes": true,
          "enable_create_root_volume": true,
          "enable_delete_existing": true
        }
      }
    }
  }

The workflow of configuration import and export consists of parts:

1. Execute workflow as in step `Import configuration`_
2. When importing succeeds, execute workflow as in step `Export configuration`_


Format of configuration data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The format to store the re-usable configuration is in JSON format and
consists of 3 sections:

* bios – ``reset`` to indicate if reset is necessary before applying settings
  indicated in the list of BIOS attribute key-value pairs inside ``settings``
  section as in Apply BIOS configuration step [3]_. If ``reset`` is false, then
  settings that are not included in ``settings`` sections are left unchanged.
* raid – as in RAID create configuration step with key-value pair settings and
  ``target_raid_config`` property [4]_
* oem – driver specific section with everything else that does not fit into
  bios and raid sections together with interface name that can handle this
  data. The interface name can be used to distinguish for which hardware type
  this configuration data is meant and used for validation during import before
  trying to parse this section and catch incompatibility early. The data format
  of this section is controlled by implementing interface and only restriction
  is that it needs to fit in JSON property.


In future new section may be added, for example, ``boot`` section to indicate
boot mode, NIC configuration for PXE boot.

There is no overlapping with ``oem`` and vendor-independent sections, like
``bios`` and ``raid``. If overlapping is determined during import, then
configuration data is considered invalid and cleaning/deployment step fails.
Depending on vendor's implementation some settings require additional logic
when determining overlapping. For example, for ``boot`` section, if that was
introduced, vendors can store these settings under OEM's BIOS section. In that
and similar cases need to check if OEM's BIOS section also contains boot
related attributes to determine overlapping when also ``boot`` section is
present.

Sample of exported data format:

.. code-block::

  {
    "bios": {
      "reset": false,
      "settings": [
        {
          "name": "name",
          "value": "value"
        },
        {
          "name": "name",
          "value": "value"
        }
      ]
    }
    "raid": {
      "create_nonroot_volumes": true,
      "create_root_volume": true,
      "delete_existing": false,
      "target_raid_config": {
        "logical_disks": [
          {
            "size_gb": 50,
            "raid_level": "1+0",
            "controller": "RAID.Integrated.1-1",
            "volume_name": "root_volume",
            "is_root_volume": true,
            "physical_disks": [
              "Disk.Bay.0:Encl.Int.0-1:RAID.Integrated.1-1",
              "Disk.Bay.1:Encl.Int.0-1:RAID.Integrated.1-1"
            ]
          },
          {
            "size_gb": 100,
            "raid_level": "5",
            "controller": "RAID.Integrated.1-1",
            "volume_name": "data_volume",
            "physical_disks": [
              "Disk.Bay.2:Encl.Int.0-1:RAID.Integrated.1-1",
              "Disk.Bay.3:Encl.Int.0-1:RAID.Integrated.1-1",
              "Disk.Bay.4:Encl.Int.0-1:RAID.Integrated.1-1"
            ]
          }
        ]
      }
    }
    "oem": {
      "interface": "idrac-redfish",
      "data": {
        "SystemConfiguration": {
          "Model": "PowerEdge R640",
          "ServiceTag": "8CY9Z99",
          "TimeStamp": "Fri Jun 26 08:43:15 2020",
          "Components": [
            {
              [...]
              "FQDD": "NIC.Slot.1-1-1",
              "Attributes": [
                {
                "Name": "BlnkLeds",
                "Value": "15",
                "Set On Import": "True",
                "Comment": "Read and Write"
                },
                {
                "Name": "VirtMacAddr",
                "Value": "00:00:00:00:00:00",
                "Set On Import": "False",
                "Comment": "Read and Write"
                },
                {
                "Name": "VirtualizationMode",
                "Value": "NONE",
                "Set On Import": "True",
                "Comment": "Read and Write"
                },
              [...]
              ]
            }
          ]
        }
    }
  }

``oem`` section of sample data depicts snippets from Dell SCP file (see more at
`idrac-redfish implementation`_) that has some metadata about the source of the
configuration (``Model``, ``ServiceTag``, ``TimeStamp``) and inside
``Components`` section there are attributes listed that can be applied during
import and is controlled by ``Set On Import`` property.

Storage of configuration data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Common functionality among hardware types is the configuration storage and will
be implemented for all vendors to be used in their implementations.

The configuration molds are stored in Swift or in web server to support
standalone Ironic and can be extended to more storage providers. New settings
to configure the location are:

.. code-block::

  [default]mold_storage
  [default]configuration_storage_dir
  [default]file_permission
  [default]http_root
  [default]http_url

The workflow for storing the configuration data:

1. Given contents for the storage to save and desired item name,
   validate the item name and if valid save the contents in a file in location
   ``[default]tempdir``. If invalid file name, step fails.
2. Based on setting ``[default]mold_storage`` determine used storage mechanism.
   If Swift is used, then upload the file to container indicated by setting
   ``[default]configuration_storage_dir``. If local web server is used, then
   upload the file to folder indicated by the same setting
   ``[default]configuration_storage_dir`` under web server root at
   ``[deploy]http_root`` and apply file permissions from
   ``[default]file_permission``. Other storage providers are out of scope.
   If there is a storage item with the same name, content is overwritten both
   in Swift and local web server filesystem.


The workflow for getting stored configuration data:

1. Given item name of the configuration and storage type in
   ``[default]mold_storage`` and location in
   ``[default]configuration_storage_dir`` fetch the data and read in memory.


idrac-redfish implementation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For iDRAC to implement these proposed steps it will use Server Configuration
Profile (SCP) [5]_ that allows to export existing configuration server and
import the same configuration file to another server. Settings for different
sub-systems such as BIOS, RAID, NIC are included in the configuration file.

The implementation would transform configuration between SCP data format and
Ironic data format. In the first version (MVP), all SCP data is exported to and
imported from ``oem`` section as-is without any transformation. In the
following versions this will be improved to start using ``bios`` and ``raid``
sections. The implementation will use Redfish protocol. As this is part of OEM
section in Redfish service, the communication will be implemented in
sushy-oem-idrac library. There is already existing functionality to work with
SCP, some additions and adjustments will be made. In next versions after MVP
is done, transformation between SCP data format and Ironic data format will be
implemented in Ironic part of idrac-redfish interface.

When comparing configuration runtime using separate BIOS and RAID configuration
jobs versus SCP approach on R640 the difference was 11 minutes versus 7 minutes
where SCP was faster within one reboot.

Alternatives
------------

We can continue to support only the current, granular hardware provisioning
deploy and clean steps.
The closest currently available functionality in Ironic is deploy templates
that enable assembling several existing steps together. In the same manner
these deploy templates can be re-used for as many systems as necessary.
However, comparing deploy templates to the proposed solution currently:

* no functionality to get the configuration from already configured system,
  user has to construct the initial configuration file themselves by hand or
  a script. To make it easier can use cached BIOS and RAID settings from a
  node that was deployed, but this re-use is still not built in Ironic.
* depending on vendor's capabilities each step may require reboot to finish.
  For example, iDRAC BIOS configuration apply needs reboot to take effect and
  deem the step to be finished. For now Ironic cannot line up several steps
  that require reboot and then finish them all by one reboot. For next step to
  start the previous one needs to be finished. The proposal makes it possible
  to handle this internally inside the import step, that is, if that is how a
  driver is implementing this, it can create 2 jobs for BIOS, RAID
  configuration and then reboot and watch for both jobs to finish to deem the
  step as finished.
* using OEM section each vendor can add support for configuring more settings
  that are not currently possible using common (vendor-independent) Ironic
  deploy steps.

This proposal does not suggest to replace current deploy steps and templates
but add alternative approach for system configuration.

Data model impact
-----------------

None

State Machine Impact
--------------------

None

REST API impact
---------------

None

Client (CLI) impact
-------------------

None

RPC API impact
--------------

None

Driver API impact
-----------------

This defines new clean and deploy steps available to all drivers from
``ManagementInterface``. These new steps are optional and do not affect drivers
which do not implement them, yet, or may never do so.

Nova driver impact
------------------

None

Ramdisk impact
--------------

None

Security impact
---------------

JSON will be used as user input. It will be validated, sanitized, and treated
as text. Common storage utils will use Python's `json.loads` when
retrieving and `json.dumps` when storing data. If there is additional
validation and clean up necessary for vendor specific implementation,
for example, OEM section, then that needs to be added to driver's
implementation.

Other end user impact
---------------------

The configuration items can accumulate in the storage as there is no default
timeout or logic that deletes them after a while because these configuration
items should be available after node's cleaning or deployment. If user do not
need the re-usable configuration items anymore, then user should delete those
themselves from the storage directly.

This adds new configuration values to ``[default]`` section to control storage
location. Default values are provided.

Scalability impact
------------------

None

Performance Impact
------------------

Depending on hardware type implementation, deployments can become faster.
When configuration storage data is processed, it is read in memory, but it is
not expected that these storage items will be large.

Also based on vendor's implementation these can be synchronous or asynchronous
steps. If steps are synchronous this will consume a long-lived thread where
operators may need to adjust the number of workers.

Other deployer impact
---------------------

None

Developer impact
----------------

There will be new clean and deploy steps available that each driver can
implement. They are optional and other developers can implement those at their
own time if needed.


Implementation
==============

Assignee(s)
-----------

Primary assignees:

* Aija Jaunteva (@ajya, aija.jaunteva@dell.com)
* Richard Pioso (@rpioso, richard.pioso@dell.com)

Other contributors:
  None

Work Items
----------

For common functionality:

* Define new clean and deployment steps in the management hardware interface
  base class, class ManagementInterface
* Implement common functionality for configuration storage
* Document new clean and deployment steps

For ``idrac-redfish`` implementation:

* Implement initial idrac hardware type derivations of the new clean and
  deployment steps which use the Redfish protocol (MVP)
* Update the iDRAC driver documentation
* Enhance the idrac hardware type implementation to support the ``bios``
  section of the configuration data
* Enhance the idrac hardware type implementation to support the ``raid``
  section of the configuration data


Dependencies
============

None


Testing
=======

For now, tempest tests are out of scope, but in future 3rd party continuous
integration (CI) tests can be added for each driver which implements the new
clean and deploy steps.


Upgrades and Backwards Compatibility
====================================

This change is designed to be backwards compatible. The new clean and deploy
steps are optional. When an attempt to use them with a hardware type which does
not implement them, then clean or deploy will fail with error saying that
node does not support these steps.


Documentation Impact
====================

* New page created under Administrator's Guide to describe new clean and deploy
  steps
* Node cleaning documentation [6]_ is updated to describe new clean steps under
  Management interface for idrac-redfish


References
==========
.. [1] https://docs.openstack.org/ironic/latest/admin/cleaning.html#cleaning-steps
.. [2] https://docs.openstack.org/ironic/latest/admin/node-deployment.html#node-deployment-deploy-steps
.. [3] https://docs.openstack.org/ironic/latest/admin/bios.html#apply-bios-configuration
.. [4] https://opendev.org/openstack/ironic/src/branch/master/ironic/drivers/raid_config_schema.json
.. [5] https://downloads.dell.com/manuals/all-products/esuprt_solutions_int/esuprt_solutions_int_solutions_resources/dell-management-solution-resources_white-papers15_en-us.pdf
.. [6] https://docs.openstack.org/ironic/latest/admin/cleaning.html
