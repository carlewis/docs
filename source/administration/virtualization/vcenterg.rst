.. _vcenterg:

======================
VMware vCenter Drivers
======================

OpenNebula seamlessly integrates vCenter virtualized infrastructures so leveraging the advanced features such as vMotion, HA or DRS scheduling provided by the VMware vSphere product family. On top of it, OpenNebula exposes a multi-tenant, cloud-like provisioning layer, including features like virtual data centers, datacenter federation or hybrid cloud computing to connect in-house vCenter infrastructures with public clouds. You also take a step toward liberating your stack from vendor lock-in.

These drivers are for companies that want to keep VMware management tools, procedures and workflows. For these companies, throwing away VMware and retooling the entire stack is not the answer. However, as they consider moving beyond virtualization toward a private cloud, they can choose to either invest more in VMware, or proceed on a tactically challenging but strategically rewarding path of open. 

The OpenNebula - vCenter combination allows you to deploy advanced provisioning infrastructures of virtualized resources. In this guide you'll learn how to configure OpenNebula to access one or more vCenters and set-up VMware-based virtual machines.

Overview and Architecture
=========================

The VMware vCenter drivers enable OpenNebula to access one or more vCenter servers that manages one or more ESX Clusters. Each ESX Cluster is presented in OpenNebula as an aggregated hypervisor, i.e. as an OpenNebula host. Note that OpenNebula scheduling decisions are therefore made at ESX Cluster level, vCenter then uses the DRS component to select the actual ESX host and Datastore to deploy the Virtual Machine.

As the figure shows, OpenNebula components see two hosts where each represents a cluster in a vCenter. You can further group these hosts into OpenNebula clusters to build complex virtual data centers for your user groups in OpenNebula.

.. note:: Together with the ESX Cluster hosts you can add other hypervisor types or even hybrid cloud instances like Microsoft Azure or Amazon EC2.

.. image:: /images/JV_architecture.png
    :width: 250px
    :align: center

Virtual Machines are deployed from VMware VM Templates that **must exist previously in vCenter**. There is a one-to-one relationship between each VMware VM Template and the equivalent OpenNebula Template. Users will then instantiate the OpenNebula Templates where you can easily build from any provisioning strategy (e.g. access control, quota...).

Networking is handled by creating Virtual Network representations of the vCenter networks. OpenNebula additionaly can handle on top of these networks three types of Address Ranges: Ethernet, IPv4 and IPv6. This networking information can be passed to the VMs through the contextualization process.

Therefore there is no need to convert your current Virtual Machines or import/export them through any process; once ready just save them as VM Templates in vCenter, following `this procedure <http://pubs.vmware.com/vsphere-55/index.jsp?topic=%2Fcom.vmware.vsphere.vm_admin.doc%2FGUID-FE6DE4DF-FAD0-4BB0-A1FD-AFE9A40F4BFE_copy.html>`__.

.. note:: After a VM Template is cloned and booted into a vCenter Cluster it can access VMware advanced features and it can be managed through the OpenNebula provisioning portal -to control the lifecycle, add/remove NICs, make snapshots- or through vCenter (e.g. to move the VM to another datastore or migrate it to another ESX). OpenNebula will poll vCenter to detect these changes and update its internal representation accordingly.

Requirements
============

The following must be met for a functional vCenter environment:

- vCenter 5.5 and/or 6.0, with at least one cluster aggregating at least one ESX 5.5 and/or 5.5 host.

- Define a vCenter user for OpenNebula. This vCenter user (let's call her oneadmin) needs to have access to the ESX clusters that OpenNebula will manage. In order to avoid problems, the hassle free approach is to declare this oneadmin user as Administrator. In production environments though, it may be needed to perform a more fine grained permission assigment (please note that the following permissions related to operations are related to the use that OpenNebula does with this operations):

+-----------------------+------------------------------------------+--------------------------------------------------+
|   vCenter Operation   |                Privileges                |                      Notes                       |
+-----------------------+------------------------------------------+--------------------------------------------------+
| CloneVM_Task          | None                                     | Creates a clone of a particular VM               |
+-----------------------+------------------------------------------+--------------------------------------------------+
| ReconfigVM_Task       | VirtualMachine.Interact.DeviceConnection | Reconfigures a particular virtual machine.       |
|                       | VirtualMachine.Interact.SetCDMedia       |                                                  |
|                       | VirtualMachine.Interact.SetFloppyMedia   |                                                  |
|                       | VirtualMachine.Config.Rename             |                                                  |
|                       | VirtualMachine.Config.Annotation         |                                                  |
|                       | VirtualMachine.Config.AddExistingDisk    |                                                  |
|                       | VirtualMachine.Config.AddNewDisk         |                                                  |
|                       | VirtualMachine.Config.RemoveDisk         |                                                  |
|                       | VirtualMachine.Config.CPUCount           |                                                  |
|                       | VirtualMachine.Config.Memory             |                                                  |
|                       | VirtualMachine.Config.RawDevice          |                                                  |
|                       | VirtualMachine.Config.AddRemoveDevice    |                                                  |
|                       | VirtualMachine.Config.EditDevice         |                                                  |
|                       | VirtualMachine.Config.Settings           |                                                  |
|                       | VirtualMachine.Config.Resource           |                                                  |
|                       | VirtualMachine.Config.AdvancedConfig     |                                                  |
|                       | VirtualMachine.Config.SwapPlacement      |                                                  |
|                       | VirtualMachine.Config.HostUSBDevice      |                                                  |
|                       | VirtualMachine.Config.DiskExtend         |                                                  |
|                       | VirtualMachine.Config.ChangeTracking     |                                                  |
|                       | VirtualMachine.Config.MksControl         |                                                  |
|                       | DVSwitch.CanUse                          |                                                  |
|                       | DVPortgroup.CanUse                       |                                                  |
|                       | VirtualMachine.Config.RawDevice          |                                                  |
|                       | VirtualMachine.Config.AddExistingDisk    |                                                  |
|                       | VirtualMachine.Config.AddNewDisk         |                                                  |
|                       | VirtualMachine.Config.HostUSBDevice      |                                                  |
|                       | Datastore.AllocateSpace                  |                                                  |
|                       | Network.Assign                           |                                                  |
+-----------------------+------------------------------------------+--------------------------------------------------+
| PowerOnVM_Task        | VirtualMachine.Interact.PowerOn          | Powers on a virtual machine                      |
+-----------------------+------------------------------------------+--------------------------------------------------+
| PowerOffVM_Task       | VirtualMachine.Interact.PowerOff         | Powers off a virtual machine                     |
+-----------------------+------------------------------------------+--------------------------------------------------+
| Destroy_Task          | VirtualMachine.Inventory.Delete          | Deletes a VM (including disks)                   |
+-----------------------+------------------------------------------+--------------------------------------------------+
| SuspendVM_Task        | VirtualMachine.Interact.Suspend          | Suspends a VM                                    |
+-----------------------+------------------------------------------+--------------------------------------------------+
| RebootGuest           | VirtualMachine.Interact.Reset            | Reboots VM's guest Operating System              |
+-----------------------+------------------------------------------+--------------------------------------------------+
| ResetVM_Task          | VirtualMachine.Interact.Reset            | Resets power on a virtual machine                |
+-----------------------+------------------------------------------+--------------------------------------------------+
| ShutdownGuest         | VirtualMachine.Interact.PowerOff         | Shutdown guest Operating System                  |
+-----------------------+------------------------------------------+--------------------------------------------------+
| CreateSnapshot_Task   | VirtualMachine.State.CreateSnapshot      | Creates a new snapshot of a virtual machine.     |
+-----------------------+------------------------------------------+--------------------------------------------------+
| RemoveSnapshot_Task   | VirtualMachine.State.RemoveSnapshot      | Removes a snapshot form a virtual machine        |
+-----------------------+------------------------------------------+--------------------------------------------------+
| RevertToSnapshot_Task | VirtualMachine.State.RevertToSnapshot    | Rever a virtual machine to a particular snapshot |
+-----------------------+------------------------------------------+--------------------------------------------------+


.. note:: For security reasons, you may define different users to access different ESX Clusters. A different user can defined in OpenNebula per ESX cluster, which is encapsulated in OpenNebula as an OpenNebula host.

- All ESX hosts belonging to the same ESX cluster to be exposed to OpenNebula **must** share one datastore among them. 

- The ESX cluster **should** have DRS enabled. DRS is not required but it is recommended. OpenNebula does not schedule to the granularity of ESX hosts, DRS is needed to select the actual ESX host within the cluster, otherwise the VM will be launched in the ESX where the VM template has been created.

- **Save as VMs Templates those VMs that will be instantiated through the OpenNebula provisioning portal**

- To enable VNC functionality for vOneCloud, repeat the following procedure for each ESX:

   - In the vSphere client proceed to Home -> Inventory -> Hosts and Clusters
   - Select the ESX host, Configuration tab and select Security Profile in the Software category.
   - In the Firewall section, select Edit. Enable GDB Server, then click OK.

.. important:: OpenNebula will **NOT** modify any vCenter configuration or manage any existing Virtual Machine.

Considerations & Limitations
============================
- **Unsupported Operations**: The following operations are **NOT** supported on vCenter VMs managed by OpenNebula, although they can be perfomed through vCenter:

+-------------+------------------------------------------------+
|  Operation  |                      Note                      |
+-------------+------------------------------------------------+
| attach_disk | Action of attaching a new disk to a running VM |
+-------------+------------------------------------------------+
| detach_disk | Action of detaching a new disk to a running VM |
+-------------+------------------------------------------------+
| migrate     | VMs cannot be migrated between ESX clusters    |
+-------------+------------------------------------------------+

- **No Security Groups**: Firewall rules as defined in Security Groups cannot be enforced in vCenter VMs.
- There is a known issue regarding **VNC ports**, preventing VMs with ID 89 to work correctly through VNC. This is being addressed `here <http://dev.opennebula.org/issues/2980>`__.
- OpenNebula treats **snapshots** a tad different from VMware. OpenNebula assumes that they are independent, whereas VMware builds them incrementally. This means that OpenNebula will still present snapshots that are no longer valid if one of their parent snapshots are deleted, and thus revert operations applied upon them will fail.
- For VNC to work properly, please install `VMware Tools (for Windows) <https://www.vmware.com/support/ws55/doc/new_guest_tools_ws.html>`__ or `Open Virtual Machine Tools <http://open-vm-tools.sourceforge.net/>`__ (for \*nix).
- **No files in context**: Passing entire files to VMs is not supported, but all the other CONTEXT sections will be honored
- Cluster name cannot contain spaces

Configuration
=============

OpenNebula Configuration
------------------------

There are two simple steps needed to configure OpenNebula so it can interact with vCenter:

**Step 1: Check connectivity**

The OpenNebula front-end needs network connectivity to all the vCenters that it is supposed to manage.

Additionaly, to enable VNC access to the spawned Virtual Machines, the front-end also needs network connectivity to all the ESX hosts

**Step 2: Enable the drivers in oned.conf**

In order to configure OpenNebula to work with the vCenter drivers, the following sections need to be uncommented or added in the ``/etc/one/oned.conf`` file:

.. code::

    #-------------------------------------------------------------------------------
    #  vCenter Information Driver Manager Configuration
    #    -r number of retries when monitoring a host
    #    -t number of threads, i.e. number of hosts monitored at the same time
    #-------------------------------------------------------------------------------
    IM_MAD = [
          name       = "vcenter",
          executable = "one_im_sh",
          arguments  = "-c -t 15 -r 0 vcenter" ]
    #-------------------------------------------------------------------------------

    #-------------------------------------------------------------------------------
    #  vCenter Virtualization Driver Manager Configuration
    #    -r number of retries when monitoring a host
    #    -t number of threads, i.e. number of hosts monitored at the same time
    #-------------------------------------------------------------------------------
    VM_MAD = [
        name       = "vcenter",
        executable = "one_vmm_sh",
        arguments  = "-t 15 -r 0 vcenter -s sh",
        type       = "xml" ]
    #-------------------------------------------------------------------------------

.. _vcenter_import_tool:

**Step 3: Importing vCenter Clusters**

OpenNebula ships with a powerful CLI tool to import vCenter clusters, VM Templates, Networks and running VMs. The tools is self-explanatory, just set the credentials and IP to access the vCenter host and follow on screen instructions. A sample section follows:

.. code::

    $ onehost list
      ID NAME            CLUSTER   RVM      ALLOCATED_CPU      ALLOCATED_MEM STAT   
                                                                                    
    $ onevcenter hosts --vcenter <vcenter-host> --vuser <vcenter-username> --vpass <vcenter-password>
    Connecting to vCenter: <vcenter-host>...done!
    Exploring vCenter resources...done!
    Do you want to process datacenter Development [y/n]? y
      * Import cluster clusterA [y/n]? y
        OpenNebula host clusterA with id 0 successfully created.

      * Import cluster clusterB [y/n]? y
        OpenNebula host clusterB with id 1 successfully created.

    $ onehost list
      ID NAME            CLUSTER   RVM      ALLOCATED_CPU      ALLOCATED_MEM STAT
       0 clusterA        -           0                  -                  - init
       1 clusterB        -           0                  -                  - init
    $ onehost list
      ID NAME            CLUSTER   RVM      ALLOCATED_CPU      ALLOCATED_MEM STAT
       0 clusterA        -           0       0 / 800 (0%)      0K / 16G (0%) on
       1 clusterB        -           0                  -                  - init
    $ onehost list
      ID NAME            CLUSTER   RVM      ALLOCATED_CPU      ALLOCATED_MEM STAT
       0 clusterA        -           0       0 / 800 (0%)      0K / 16G (0%) on
       1 clusterB        -           0      0 / 1600 (0%)      0K / 16G (0%) on


The following variables are added to the OpenNebula hosts representing ESX clusters:

+------------------+------------------------------------+
|    Operation     |                Note                |
+------------------+------------------------------------+
| VCENTER_HOST     | hostname or IP of the vCenter host |
+------------------+------------------------------------+
| VCENTER_USER     | Name of the vCenter user           |
+------------------+------------------------------------+
| VCENTER_PASSWORD | Password of the vCenter user       |
+------------------+------------------------------------+

.. note::

   OpenNebula will create a special key at boot time and save it in /var/lib/one/.one/one_key. This key will be used as a private key to encrypt and decrypt all the passwords for all the vCenters that OpenNebula can access. Thus, the password shown in the OpenNebula host representing the vCenter is the original password encrypted with this special key.

.. _import_vcenter_resources:

**Step 4: Importing vCenter VM Templates, Networks and running VMs**

The same **onevcenter** tool can be used to import existing VM templates from the ESX clusters:

.. code::

    $ ./onevcenter templates --vcenter <vcenter-host> --vuser <vcenter-username> --vpass <vcenter-password>

    Connecting to vCenter: <vcenter-host>...done!

    Looking for VM Templates...done!

    Do you want to process datacenter Development [y/n]? y

      * VM Template found:
          - Name   : ttyTemplate
          - UUID   : 421649f3-92d4-49b0-8b3e-358abd18b7dc
          - Cluster: clusterA
        Import this VM template [y/n]? y
        OpenNebula template 4 created!

      * VM Template found:
          - Name   : Template test
          - UUID   : 4216d5af-7c51-914c-33af-1747667c1019
          - Cluster: clusterB
        Import this VM template [y/n]? y
        OpenNebula template 5 created!

    $ onetemplate list
      ID USER            GROUP           NAME                                REGTIME
       4 oneadmin        oneadmin        ttyTemplate                  09/22 11:54:33
       5 oneadmin        oneadmin        Template test                09/22 11:54:35

    $ onetemplate show 5
    TEMPLATE 5 INFORMATION
    ID             : 5
    NAME           : Template test
    USER           : oneadmin
    GROUP          : oneadmin
    REGISTER TIME  : 09/22 11:54:35

    PERMISSIONS
    OWNER          : um-
    GROUP          : ---
    OTHER          : ---

    TEMPLATE CONTENTS
    CPU="1"
    MEMORY="512"
    PUBLIC_CLOUD=[
      TYPE="vcenter",
      VM_TEMPLATE="4216d5af-7c51-914c-33af-1747667c1019" ]
    SCHED_REQUIREMENTS="NAME=\"devel\""
    VCPU="1"

Moreover the same **onevcenter** tool can be used to import existing Networks and distributed vSwitches from the ESX clusters:

.. code::

    $ .onevcenter networks --vcenter <vcenter-host> --vuser <vcenter-username> --vpass <vcenter-password>

    Connecting to vCenter: <vcenter-host>...done!

    Looking for vCenter networks...done!

    Do you want to process datacenter vOneDatacenter [y/n]? y

      * Network found:
          - Name    : MyvCenterNetwork
          - Type    : Port Group
        Import this Network [y/n]? y
        How many VMs are you planning to fit into this network [255]? 45
        What type of Virtual Network do you want to create (IPv[4],IPv[6],[E]thernet) ? E
        Please input the first MAC in the range [Enter for default]:
        OpenNebula virtual network 29 created with size 45!

        $ onevnet list
          ID USER            GROUP        NAME                CLUSTER    BRIDGE   LEASES
          29 oneadmin        oneadmin     MyvCenterNetwork    -          MyFakeNe      0

        $ onevnet show 29
        VIRTUAL NETWORK 29 INFORMATION
        ID             : 29
        NAME           : MyvCenterNetwork
        USER           : oneadmin
        GROUP          : oneadmin
        CLUSTER        : -
        BRIDGE         : MyvCenterNetwork
        VLAN           : No
        USED LEASES    : 0

        PERMISSIONS
        OWNER          : um-
        GROUP          : ---
        OTHER          : ---

        VIRTUAL NETWORK TEMPLATE
        BRIDGE="MyvCenterNetwork"
        PHYDEV=""
        VCENTER_TYPE="Port Group"
        VLAN="NO"
        VLAN_ID=""

        ADDRESS RANGE POOL
         AR TYPE    SIZE LEASES               MAC              IP          GLOBAL_PREFIX
          0 ETHER     45      0 02:00:97:7f:f0:87               -                      -

        LEASES
        AR  OWNER                    MAC              IP                      IP6_GLOBAL

Also, the **onevcenter** tool can be used to import existing running VMs:

.. code::

    $ .onevcenter vms --vcenter <vcenter-host> --vuser <vcenter-username> --vpass <vcenter-password>

    Connecting to vCenter: <vcenter-host>...done!

    Looking for running Virtual Machines...done!

    Do you want to process datacenter Testing [y/n]? y

      * Running Virtual Machine found:
          - Name   : vCenterTesting
          - UUID   : 42246fa6-340e-5801-4598-c65672dfda64
          - Cluster: Testing
        Import this Virtual Machine [y/n]? y
        OpenNebula VM 11 created!

After a Virtual Machine is imported, their lifecycle (including creation of snapshots) can be controlled through OpenNebula. The following operations *cannot* be performed on an imported VM:

- Delete --recreate
- Undeploy (and Undeploy --hard)
- Migrate (and Migrate --live)
- Stop

Also, network management operations are present like the ability to attach/detach network interfaces.

.. _reacquire_vcenter_resources:

The same import mechanism is available graphically through Sunstone for hosts, networks, templates and running VMs, using the vCenter host create dialog, and they can be used to reacquire resources after the host has been created.

.. image:: /images/vcenter_create.png
    :width: 90%
    :align: center

.. note:: running VMS can only be imported after the vCenter host has been successfuly acquired. VMs will appear in the Pending state in vOneCloud until the scheduler automatically passes them to Running, there is no need to force the deployment.

.. note:: If you are running Sunstone using nginx/apache you will have to forward the following headers to be able to interact with vCenter, HTTP_X_VCENTER_USER, HTTP_X_VCENTER_PASSWORD and HTTP_X_VCENTER_HOST. For example in nginx you have to add the following attrs to the server section of your nginx file (underscores_in_headers on; proxy_pass_request_headers on;)

Usage
=====

.. _vm_template_definition_vcenter:

VM Template definition
----------------------

In order to manually create a VM Template definition in OpenNebula that represents a vCenter VM Template, the following attributes are needed:

+--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|     Operation      |                                                                                                   Note                                                                                                  |
+--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| CPU                | Physical CPUs to be used by the VM. This **must** relate to the CPUs used by the vCenter VM Template                                                                                                    |
+--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| MEMORY             | Physical CPUs to be used by the VM. This **must** relate to the CPUs used by the vCenter VM Template                                                                                                    |
+--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| NIC                | Check :ref:`VM template reference <template_network_section>`. Valid MODELs are: virtuale1000, virtuale1000e, virtualpcnet32, virtualsriovethernetcard, virtualvmxnetm, virtualvmxnet2, virtualvmxnet3. |
+--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| GRAPHICS           | Multi-value - Only VNC supported, check the  :ref:`VM template reference <io_devices_section>`.                                                                                                         |
+--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| PUBLIC_CLOUD       | Multi-value. TYPE must be set to vcenter, and VM_TEMPLATE must point to the uuid of the vCenter VM that is being represented                                                                            |
+--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| SCHED_REQUIREMENTS | NAME="name of the vCenter cluster where this VM Template can instantiated into a VM". See :ref:`VM Scheduling section <vm_scheduling_vcenter>` for more details.                                        |
+--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| CONTEXT            | All :ref:`sections <template_context>` will be honored except FILES                                                                                                                                     |
+--------------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

You can find more information about contextualization in the :ref:`vcenter Contextualization <vcenter_context>` section.

After a VM Template is instantiated, the lifecycle of the resulting virtual machine (including creation of snapshots) can be controlled through OpenNebula. Also, network management operations are present like the ability to attach/detach network interfaces.

.. _virtual_network_vcenter_usage:

Virtual Network definition
--------------------------

Virtual Networks from vCenter can be represented using OpenNebula standard networks, taking into account that the BRIDGE of the Virtual Network needs to match the name of the Network defined in vCenter. OpenNebula supports both "Port Groups" and "Distributed Port Groups".

Virtual Networks in vCenter can be created using the vCenter web client, with any specific configuration like for instance VLANs. OpenNebula will use these networks with the defined characteristics, but it cannot create new Virtual Networks in vCenter, but rather only OpenNebula vnet representations of such Virtual Networks. OpenNebula additionaly can handle on top of these networks three types of :ref:`Address Ranges: Ethernet, IPv4 and IPv6 <vgg_vn_ar>`.

vCenter VM Templates can define their own NICs, which OpenNebula cannot manage. However, any NIC added in the OpenNebula VM Template, or through the attach_nic operation, will be handled by OpenNebula, and as such it is subject to be detached and its informatin (IP, MAC, etc) is known by OpenNebula.

.. _vm_scheduling_vcenter:

VM Scheduling
-------------

OpenNebula scheduler should only chose a particular OpenNebula host for a OpenNebula VM Template representing a vCenter VM Template, since it most likely only would be available in a particular vCenter cluster.

Since a vCenter cluster is an aggregation of ESX hosts, the ultimate placement of the VM on a particular ESX host would be managed by vCenter, in particular by the `Distribute Resource Scheduler (DRS) <https://www.vmware.com/es/products/vsphere/features/drs-dpm>`__.

In order to enforce this compulsory match between a vCenter cluster and a OpenNebula/vCenter VM Template, add the following to the OpenNebula VM Template:

.. code::

    SCHED_REQUIREMENTS = "NAME=\"name of the vCenter cluster where this VM Template can instantiated into a VM\""

In Sunstone, a host abstracting a vCenter cluster will have an extra tab showing the ESX hosts that conform the cluster.

.. image:: /images/host_esx.png
    :width: 90%
    :align: center

VM Template Cloning Procedure
=============================

OpenNebula uses VMware cloning VM Template procedure to instantiate new Virtual Machines through vCenter. From the VMware documentation:

-- Deploying a virtual machine from a template creates a virtual machine that is a copy of the template. The new virtual machine has the virtual hardware, installed software, and other properties that are configured for the template.

A VM Template is tied to the host where the VM was running, and also the datastore(s) where the VM disks where placed. Due to shared datastores, vCenter can instantiate a VM Template in any of the hosts beloning to the same cluster as the original one. 

OpenNebula uses several assumptions to instantitate a VM Template in an automatic way:

- **diskMoveType**: OpenNebul instructs vCenter to "move only the child-most disk backing. Any parent disk backings should be left in their current locations.". More information `here <https://www.vmware.com/support/developer/vc-sdk/visdk41pubs/ApiReference/vim.vm.RelocateSpec.DiskMoveOptions.html>`__

- Target **resource pool**: OpenNebula uses the default cluster resource pool to place the VM instantiated from the VM template
