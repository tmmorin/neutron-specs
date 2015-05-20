..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Extension for BGP VPN
==========================================

https://blueprints.launchpad.net/neutron/+spec/neutron-bgp-vpn


Problem Description
===================

BGP-based L3VPN networks are widely used in the industry especially for enterprises.
We are going to support inter-connection between L3VPNs & Neutron networks.

A typical use-case is the following: there are yellow tenant and green tenant.
Both tenants have a VPN (a set of external sites), and they are going to trigger the
connection of one or more Neutron networks to their sites using BGP-based L3VPN.

Another use case is when E-VPN is used to provide an Ethernet interconnect between
multiple sites.


Proposed Change
===============

The Blueprint defines a set of APIs required to interconnect a OpenStack virtual-network
with an L3VPN network as defined by RFC4364 (BGP/MPLS Virtual Private Networks).
The framework is made generic to also support E-VPN.

Alternatives and related techniques
-----------------------------------

Other techniques are available to build VPNs, but the goal of this proposal is to
make it possible to create the interconnect we need when the technology of BGP-based VPNs
is already used oustide an Openstack cloud.

There is also related blueprint for BGP dynamic routing (see refs).
But, while the goal of this blueprint is to provide tenant facing endpoints for BGPVPN,
the goal of BGP dynamic routing is provide resource model for
administrators.

Data Model Impact
-----------------

We are going to add new BGPVPNConnection resource which builds a relationship between a
set of Neutron networks and a set of parameters for a BGP-based VPN.  A BGPVPNConnection
is created by the admin and given to a tenant who can then associate it to Networks.

Among these parameters a key one is the Route Targets. In BGP-based VPNs, a set of identifiers
called Route Targets are associated with a VPN, and in the typical case identify a VPN ; they
can also be used to build other VPN topologies such as hub'n'spoke.

Here is a conceptual model relationship with BGPVPN terminology. Each VRF (Virtual Routing and
Forwarding) in PE imports route from Route Targets, and export routes to Route Targets. If
a VRF imports from a Route Target, BGP VPN IP routes will be imported in this VRF.
If a VRF exports to a Route Target, the routes in the VRF will be associated to this Route Target
and announced by BGP.


REST API Impact
---------------

This is a description of a BGPVPNConnection resource.

.. csv-table:: BGPVPNConnection
    :header: Attribute Name,Type,Access,Default Value,Validation/Constraint,Description

    id,uuid-str,R,Generated,UUID_PATTERN,id of BGPVPNConnection resource
    name,string,RW,"",String,name of this connection
    tenant_id,uuid-str,R,,UUID_PATTERN,tenant_id of owner
    networks,list(uuid-str),RW,None,N/A,List of ids of the networks associated with this BGPVPNConnection
    type, string, R, l3, for instance "l3 or l2", selection of the type of VPN and the technology behind it
    route_targets,list(str),RW admin only,N/A,List of valid route-target strings (see below),Route Targets that will be both imported and use for export
    import_targets,list(str),RW admin only,N/A,List of valid route-target strings (see below),addtional Route Targets that will be imported
    export_targets,list(str),RW admin only,N/A,List of valid route-target strings (see below),additional Route Targets that will be used for export
    route_distinguishers,list(str),RW admin only,None,List of valid route-distinguisher strings (see below),(if this parameter is specified) one of these RDs will be used to advertize VPN routes
    auto_aggregate,bool,RW admin only,False,{ True | False },enable aggreation or not (type l3 only)

**FIXME** Explain type, how it describes the technology that can be used

On Route Targets:

* The set of RTs used for import is the union of route_targets and import_targets.
* The set of RTs used for export is the union of route_targets and export_targets.
* One of route_target, import_target or export_target options must be defined.

For intance, in the very typical use case where the BGPVPNConnection uses a single Route Target for both
import and export, the route_targets parameter alone is enough and will contain one Route target.

The route_distinguishers parameter is optional and provides an indication of
the RDs that shall be used for routes announced for Neutron networks.
When specified, the implementation will pick, for a said iadvertisement, one of these RDs.
An implementation may or may not support this behavior, and should report an API error in the later case.
When not specified, the implementation will use automatically-assigned RDs (for instance <ip>:<number>
RDs derived from the PE IP).

Valid strings for Route Targets and Route Distinguisher are the following:

* <2-byte AS#>:<32bit-number>
* <4-byte AS#>:<16bit-number>
* <4-byte IPv4>:<16bit-number>

The auto-aggregate flag controls whether or not routes should be automatically aggregated
before being advertised outside Neutron.
An implementation may or may not support this behavior, and should report an API error in the later case.

**FIXME** TBC API for listing the supported types

Security Impact
---------------

BGPVPNConnection impacts external connectivity and controle the isolation between VPNs, because of
this VPN parameters cannot be chosen by tenants. In addition, network operators may prefer to not expose actual Route Target value for the users.

So there are two workflows, one for the admin, one for a tenant.

Admin/Operator Workflow: Setup of a BGPVPN connection

* the cloud/network admin creates a BGPVPNConnection for a tenant based on contract and OSS information about the VPN for this tenant
* at this stage, the list of associated Networks can be empty

Tenant Workflow: Association of a BGPVPNConnection and Networks, on-demand

* the tenant lists the BGPVPNConnection that he can use
* the tenant associates a BGPVPNConnection with one or more Networks.

Notifications Impact
--------------------

A Service plug-in should send CRUD event notification of the BGPVPNConnection.

**FIXME**: what does this mean ?

Other End User Impact
---------------------

We are also going to add support for this in python-neutronclient.
Here is a list of command we will have

::

    # Admin
    neutron bgpvpn-connection-create --route-target list=true 64512:1,64512:2
    (returns info on the BGPVPNConnection created)

    # Tenant
    neutron bgpvpn-connection-list
    neutron bgpvpn-connection-update <bgpvpnconnection-uuid> --network-id list=true <network-uuid>
    neutron bgpvpn-connection-delete <bgpvpnconnection-uuid>


Performance Impact
------------------

BGPVPNConnection table will have a table relationship for Network.

Other Deployer Impact
---------------------

Current reference implementation plan is to use Bagpipe BGP
However, it isn't limited if there is another option.

Developer Impact
----------------

* We will have a database table for BGPVPNConnection and a proper migration scripts.
* Reference implementation will use Bagpipe BGP

Community Impact
----------------

N/A

IPv6 Impact
-----------

This API is compatible with IPv6 route advertisement/discovery.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

* Mathieu Rohon <matrohon>

Other contributors:

* Thomas Morin
* Vikram Choudhary
* Diego Garcia del Rio
* Peter V Saveliev
* Robert Raszuk
* Nachi Ueno
* Pedro Marques


Work Items
----------

- BGPVPNConnection API Extension
- Framework to do DB persistency and load the selected Service driver
- python-neutronclient plugin to use this API
- driver for Bagpipe BGP
- driver for OpenDaylight
- driver for OpenContrail
- driver for Nuage Networks


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

If relevant additionally to Tempest tests.

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

* RFC4364 BGP/MPLS IP Virtual Private Networks (IP VPNs) http://tools.ietf.org/html/rfc4364
* RFC7432 BGP MPLS-Based Ethernet VPN (Ethernet VPNs, a.k.a E-VPN) http://tools.ietf.org/html/rfc7432
* BGPVPN stackforge project : http://git.openstack.org/cgit/stackforge/networking-bgpvpn
* Telco WorkGroupe use case : https://review.openstack.org/#/c/171680/
* OpenContrail plug-in :doc`:opencontrail-plugin.rst`
* BGP dynamic routing : https://blueprints.launchpad.net/neutron/+spec/bgp-dynamic-routing
* Bagpipe BGP speaker : https://github.com/Orange-OpenSource/bagpipe-bgp
