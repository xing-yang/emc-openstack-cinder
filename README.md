Introduction
============

This guide explains how to configure and make use of the EMC Cinder driver with the Folsom release of OpenStack.


Overview
========

The EMC Cinder driver is based on the existing ISCSIDriver, with the ability to create/delete and attach/detach volumes and create/delete snapshots, etc.

The Cinder driver executes the volume operations by communicating with the backend EMC storage. It uses a CIM client in python called PyWBEM to make CIM operations over HTTP.

The EMC CIM Object Manager (ECOM) is packaged with the EMC SMI-S Provider. It is a CIM server that allows CIM clients to make CIM operations over HTTP, using SMI-S in the backend for EMC storage operations.

The EMC SMI-S Provider supports the SNIA Storage Management Initiative (SMI), an ANSI standard for storage management. It supports VMAX/VMAXe and VNX storage systems.


Requirements
============

EMC SMI-S Provider V4.5 and higher is required.  SMI-S can be downloaded from EMC's Powerlink web site http://powerlink.emc.com.  Refer to the EMC SMI-S Provider release notes for installation instructions. 

VMAX/VMAXe arrays and VNX arrays are supported.


Supported Operations
====================

The following operations will be supported on both VMAX/VMAXe and VNX arrays:
* Create volume
* Delete volume
* Attach volume
* Detach volume

Snapshot is not supported on VMAXe with SMI-S 4.5.0.  Support will be added in SMI-S 4.5.1.  The following operations will be supported on VMAX and VNX arrays.
* Create snapshot
* Delete snapshot

The following operations will be supported on VNX only:
* Create volume from snapshot

Only thin provisioning is supported by the EMC Cinder driver.


Preparation
===========

Download Cinder driver
----------------------

Download the EMC Cinder driver file from the following location: https://github.com/june123/emc-openstack-cinder/blob/master/emc.py

Copy emc.py to the cinder/volume directory of your OpenStack node(s) where cinder-volume is running.  This directory is where other Cinder drivers are located.

Install PyWBEM on the Openstack node where you copied Cinder driver emc.py:
```
$ sudo apt-get install python-pywbem
```

Setup SMI-S
-----------

Download SMI-S 4.5.0 from PowerLink and install it following the instructions of SMI-S release notes.  Add your VNX/VMAX/VMAXe arrays to SMI-S following the SMI-S release notes. 

Register with VNX
-----------------

For a VNX volume to be exported to a Compute node, the node needs to be registered with VNX first.

On the Compute node 1.1.1.1, do the following (assume 10.10.61.35 is the iscsi target):
```
$ sudo /etc/init.d/open-iscsi start
$ sudo iscsiadm -m discovery -t st -p 10.10.61.35
$ cd /etc/iscsi
$ sudo more initiatorname.iscsi
$ iscsiadm -m node
```

Log in to VNX from the Compute node using the target corresponding to the SPA port:
```
$ sudo iscsiadm -m node -T iqn.1992-04.com.emc:cx.apm01234567890.a0 -p 10.10.61.35 -l
```

Assume "iqn.1993-08.org.debian:01:1a2b3c4d5f6g" is the initiator name of the Compute node.  Login to Unisphere, go to VNX00000->Hosts->Initiators, Refresh and wait until initiator "iqn.1993-08.org.debian:01:1a2b3c4d5f6g" with SP Port "A-8v0" appears.

Click the "Register" button, select "CLARiiON/VNX" and enter the host name and IP address:
```
myhost1
1.1.1.1
```

Click Register. Now host 1.1.1.1 will appear under Hosts->Host List as well.

Log out of VNX on the Compute node:
```
$ sudo iscsiadm -m node -u
```

Log in to VNX from the Compute node using the target corresponding to the SPB port:
```
$ sudo iscsiadm -m node -T iqn.1992-04.com.emc:cx.apm01234567890.b8 -p 10.10.10.11 -l
```

In Unisphere register the initiator with the SPB port.

Log out:
```
$ sudo iscsiadm -m node -u
```

Create Masking View on VMAX/VMAXe
---------------------------------

For VMAX/VMAXe, user needs to do initial setup on the SMC server first.  On the SMC server, create initiator group, storage group, port group, and put them in a masking view.  Initiator group contains the initiator names of the openstack hosts.  Storage group should have at least 6 gatekeepers.


Setup
=====

cinder.conf
-----------

Make the following changes in /etc/cinder/cinder.conf:

Comment out the root_helper flag in cinder.conf.  ``root_helper = sudo`` by default.

For VMAX/VMAXe, we have the following entries where 10.10.61.45 is the IP address of the VMAX/VMAXe iscsi target.
```
iscsi_target_prefix = iqn.1992-04.com.emc
iscsi_ip_address = 10.10.61.45
volume_driver = cinder.volume.emc.EMCISCSIDriver
cinder_emc_config_file = /etc/cinder/cinder_emc_config.xml
```

For VNX, we have the following entries where 10.10.61.35 is the IP address of the VNX iscsi target.
```
iscsi_target_prefix = iqn.2001-07.com.vnx
iscsi_ip_address = 10.10.61.35
volume_driver = cinder.volume.emc.EMCISCSIDriver
cinder_emc_config_file = /etc/cinder/cinder_emc_config.xml
```
Restart the cinder-volume service.

cinder_emc_config.xml								---------------------

Create the file /etc/cinder/cinder_emc_config.xml.  We don't need to restart service for this change.
										For VMAX/VMAXe, we have the following in the xml file:
```
<?xml version='1.0' encoding='UTF-8'?>
<EMC>
<StorageType>xxxx</StorageType>
<MaskingView>xxxx</MaskingView>
<EcomServerIp>x.x.x.x</EcomServerIp>
<EcomServerPort>xxxx</EcomServerPort>
<EcomUserName>xxxxxxxx</EcomUserName>
<EcomPassword>xxxxxxxx</EcomPassword>
</EMC>
```

For VNX, we have the following in the xml file:
```
<?xml version='1.0' encoding='UTF-8'?>
<EMC>
<StorageType>xxxx</StorageType>
<EcomServerIp>x.x.x.x</EcomServerIp>
<EcomServerPort>xxxx</EcomServerPort>
<EcomUserName>xxxxxxxx</EcomUserName>
<EcomPassword>xxxxxxxx</EcomPassword>
</EMC>
```

MaskingView is required for attaching VMAX/VMAXe volumes to an OpenStack VM.  A Masking View can be created using SMC.  The Masking View needs to have an Initiator Group that contains the initiator of the OpenStack compute node that hosts the VM.

StorageType is the thin pool where user wants to create volume from.  Only thin LUNs are supported by the plugin.  <StorageType> is required for both VMAX/VMAXe and VNX.  Thin pools can be created using SMC for VMAX/VMAXe and Unisphere for VNX.

EcomServerIp and EcomServerPort are the IP address and port number of the ECOM server which is packaged with SMI-S.  EcomUserName and EcomPassword are credentials for the ECOM server.


``Copyright (c) 2012 EMC Corporation.``
