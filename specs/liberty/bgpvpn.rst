..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Neutron extension for BGP VPN
==========================================

https://blueprints.launchpad.net/neutron/+spec/neutron-bgp-vpn


Problem Description
===================

L3VPN network is widely used in industry especially for enterprise.
We are going to support inter connection between L3VPN & Neutron networks.

This is a usecase 1. In this usecase, there are yellow tenant and green tenant.
Both tenants have an external site, and they are going to connect a neutron network for
their site using L3VPN or eVPN.


Proposed Change
===============

The Blueprint defines a set of APIs required to interconnect a OpenStack virtual-network
with an L3VPN network as defined by RFC4364 (BGP/MPLS Virtual Private Networks).

Alternatives
------------

IPsec VPN and SSL-VPNs are vpn alternatives. However, they can't fix large scale
enterprise networking who need dynamic routing in multiple data centers.

There is also related blueprint for BGP dynamic routing (see refs).
The goal of this blueprint is to provide tenant facing endpoints for BGPVPN
while the goal of BGP dynamic routing is provide resource model for
administrators.

Data Model Impact
-----------------

We are going to add new BGPVPNConnection resources.
BGPVPNConnection will have reference to Network. In BGPVPN, we identify a
vpn using route target identifier. BGPVPNConnection is a relationship with
Neutron and route target.

Deletion for associated network should be prohibited by a proper exception.

Here is a conceptual model relationship with BGPVPN terminology. VRFs (Virtual Routing and Forwarding) in
PE can import/export route target. If a VRF imports route target, associated route will be imported.
If a VRF exports route target, route in the VRF will be associated and exported with route target.
Same thing will happen for the Neutron network.

REST API Impact
---------------

This is a description of BGPVPNConnection.
One of route_target, import_target or export_target options must be defined.
The common use case is for BGP VPN connectivity to use a single bi-directional route-target.
However it is possible to have the auto-aggregate flag control.
This flag controls whether or not routes should be automatically aggregated
when advertised to an external network.

.. csv-table:: BGPVPNConnection
    :header: Attribute Name,Type,Access,Default Value,Validation/Constraint,Description

    id,uuid-str,R,Generated,UUID_PATTERN,id of BGPVPNConnection resource
    name,string,RW,"",String,name of this connection
    tenant_id,uuid-str,R,,UUID_PATTERN,tenant_id of owner
    network_id,uuid-str,RW,None,N/A,network id associated with route target
    type, string, R, l3, "l3 or l2", selection for evpn or l3vpn
    route_target,list(str),RW admin only,N/A,List of valid route-target strings.<as#>:32bit-number,route target will be import/export
    import_target,list(str),RW admin only,N/A,List of valid route-target strings.<as#>:32bit-number,route targets will be imported
    export_target,list(str),RW admin only,N/A,List of valid route-target strings.<as#>:32bit-number,route targets will be exported
    auto_aggregate,bool,RW admin only,TRUE,{ True | False },enable aggreation or not (l3vpn only)

We will have a database table for BGPVPNConnection and a proper migration scripts.


Security Impact
---------------

BGPVPNConnection impacts external connectivity. In addition, network operators
don't want expose actual route target value for the users.
so there could be two work flows.

Workflow A: Operator will setup BGPVPN connection

* Network operator creates BGPVPNConnection for a tenant based on contract with specified network

Workflow B: Tenant will associate BGPVPN and Network on-demand

* Network operator creates BGPVPNConnection for a tenant based on contract. Network id can be None in this stage.
* A tenant will associate BGPVPNConnection with a network.

Notifications Impact
--------------------

A Service plug-in should send CRUD event notification of the BGPVPNConnection.

Other End User Impact
---------------------

We are also going to add support for this in python-neutronclient.
Here is a list of command we will have

::

    # Admin
    neutron bgpvpn-connection-create --route-target list=true 64512:1,64512:2

    # Tenant
    neutron bgpvpn-connection-update <bgpvpn id> --network-id <uuid>
    neutron bgpvpn-connection-list
    neutron bgpvpn-connection-delete <bgpvpn id>


Performance Impact
------------------

BGPVPNConnection table will have a table relationship for Network.
Especially speaking, we need additional check when we try to delete network.

Other Deployer Impact
---------------------

Current Reference implementation plan is to use Bagpipe BGP speaker.
However, it isn't limited if there is another option.

Developer Impact
----------------

* Reference implementation will use Bagpipe BGP

Community Impact
----------------

N/A

IPv6 Impact
-----------

Bagpipe is compatible with IPv6 route advertisement/discovery

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Mathieu Rohon <matrohon>

Other contributors:
    Thomas Morin
    Nati Ueno
    Pedro Marques


Work Items
----------

- L3VPNConnection API Extension
- Framework to load selected Service driver
- Bagpipe BGP support
- OpenDaylight support
- OpenContrail support


Dependencies
============

* BagPipe for the reference implementation


Testing
=======

API Tests
---------

Unit tests for CRUD operations

Tempest Tests
-------------

Connection between two sites

Functional Tests
----------------


Documentation Impact
====================

User Documentation
------------------

The API should be documented, along with the way to
configure neutron to use the BGPVPN Service Plugin

Developer Documentation
-----------------------

The developer documentation should mention how to interact
with the bgp vpn service framework


References
==========

* RFC4364 BGP/MPLS IP Virtual Private Networks (VPNs) http://tools.ietf.org/html/rfc4364
* BGPVPN stackforge project : http://git.openstack.org/cgit/stackforge/networking-bgpvpn
* Telco WorkGroupe use case : https://review.openstack.org/#/c/171680/
* OpenContrail plug-in :doc`:opencontrail-plugin.rst`
* BGP dynamic routing : https://blueprints.launchpad.net/neutron/+spec/bgp-dynamic-routing
* Bagpipe BGP speaker : https://github.com/Orange-OpenSource/bagpipe-bgp
