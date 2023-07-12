---
stand_alone: true
category: std
submissionType: IETF
ipr: trust200902
lang: en

title: Edge-to-edge Encryption in Multi-segment SD-WAN
abbrev: e2e encryption SDWAN
docname: draft-sheng-idr-e2e-encryption-in-sd-wan-latest
obsoletes:
updates:
# date: 2022-02-02 -- date is filled in automatically by xml2rfc if not given

area: Routing
workgroup: idr

kw:
  - Multi-segment SD-WAN
  - IPSec

author:
 -
  ins: C. Sheng
  name: Cheng Sheng
  organization: Huawei
  street: Beiqing Road
  city: Beijing
  email: shengcheng@huawei.com
 -
  ins: H. Shi
  name: Hang Shi
  organization: Huawei
  email: shihang9@huawei.com
  role: editor
  street: Beiqing Road
  city: Beijing
  country: China

informative:
  I-D.draft-dmk-rtgwg-multisegment-sdwan:

--- abstract

The document describes the control plane enhancement for multi-segment SD-WAN to implement Edge-to-edge encryption, GW information exchange, include/exclude transit list exchange.

--- middle

# Introduction {#intro}

To interconnect geographically faraway branches or SASE resources, multi-segment SD-WAN is often deployed via cloud backbone{{I-D.draft-dmk-rtgwg-multisegment-sdwan}}. This document focuses on the scenario where the edge connects to the Cloud GW through unsafe public network such as internet, as shown in {{Scenario}}. IPSec encryption is used to provide the authentication, integrity and confidentiality of the data from edge to edge.

~~~
  (^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^)
  (            Region2             )
  (            +-----+             )
  (            | GW2 |             )
  (            +-----+             )
  (           /       \            )
  (          /  Cloud  \           )
  (         /  Backbone \          )
  ( Region1/             \Region3  )
  ( +-----+               +-----+  )
  ( | GW1 |---------------| GW3 |  )
  ( +--+--+               +--+--+  )
  (^^^^|^^^^^^^^^^^^^^^^^^^^^|^^^^^)
       |                     |
       | <-Public Internet-> |
    +--+--+                +--+--+
    |CPE 1|                |CPE 2|
    +-----+                +-----+
~~~
{: #Scenario  title="Multi-segment SD-WAN"}

To reduce the computing overhead of the Cloud GW, it is preferred to establish IPSec tunnel from edge to edge rather than from edge to Cloud GW. The overlay routing information is carried outside of the encrypted payload. When GW receives packet from CPE, it only needs to look at the unencrypted overlay routing header to do the forwarding. This document describes the control plane enhancement on top of {{!I-D.draft-ietf-idr-sdwan-edge-discovery}} to exchange the IPSec SA and corresponding GW information between edges to enable edge-to-edge IPSec encryption. This document also defines an extension to exchange the include/exclude transit list information from egress edge to ingress edge.

# Requirements Language

{::boilerplate bcp14-tagged}

# Exchange IPSec SA for edge-to-edge encryption {#IPSEC}

All CPEs are under the control of one BGP instance. {{!I-D.draft-ietf-idr-sdwan-edge-discovery}} defines the mechanism for SD-WAN edges to discover each other's properties. The IPSec Key exchange between the CPE is via BGP update through RR. However, the granularity of IPSec SA in {{!I-D.draft-ietf-idr-sdwan-edge-discovery}} is limited to per site, per node or per port and it does not specify if the IPSec is between edge to edge or edge to Cloud GW. To that end, this document defines a new route type to exchange the IPSec SA for edge-to-edge IPSec encryption. The format is shown in {{e2e-route-type}}.

~~~
               +---------------------+
               |  Route Type = TBD   | 2 octet
               +---------------------+
               |       Length        | 2 Octet
               +---------------------+
               | Route-Distinguisher | 8 octets
               +---------------------+
               |     SD-WAN-Color    | 4 octets
               +---------------------+
               |    SD-WAN-Node-ID   | 4 or 16 octets
               +---------------------+
~~~
{: #e2e-route-type  title="Edge-to-edge IPSec encryption NLRI"}

where:

- NLRI Route Type = TBD: For advertising edge-to-edge IPSec encryption where the IPSec SA can be uniquely identified by a tuple: &lt;Route-Distinguisher, SD-WAN-Color, SD-WAN-Node-ID&gt;
- Route-Distinguisher: an 8-octet value used to distinguish routes from different VPN (see {{!RFC4364}}).
- SD-WAN-Color: represent the SD-WAN site ID.
- SD-WAN-Node-ID: The node's IPv4 or IPv6 address.

The IPSec SA related sub-TLVs remain the same as defined in {{!I-D.draft-ietf-idr-sdwan-edge-discovery}} and is carried in the SD-WAN-Hybrid Tunnel Encapsulation attribute.

# Exchange corresponding GW information between edges

To help the ingress Cloud GW(GW1) route and forward to the egress Cloud GW, the source CPE need to carry the egress GW information in the data plane. To that end, the CPE need to discover the corresponding GW and advertise the corresponding GW information to other CPEs. Note that there may exist multiple path between the CPE and the corresponding GW. The source CPE may need to choose a specific path. This document defines a new NLRI route-type to carry the corresponding GW information. The format is shown in {{GW-info-route-type}}.

~~~
               +---------------------+
               |  Route Type = TBD   | 2 octet
               +---------------------+
               |       Length        | 2 octet
               +---------------------+
               |     SD-WAN-Color    | 4 octets
               +---------------------+
               |    SD-WAN-Node-ID   | 4 or 16 octets
               +---------------------+
~~~
{: #GW-info-route-type  title="Corresponding GW information NLRI"}

where: the SD-WAN-Color and SD-WAN-Node-ID is the same as in {{IPSEC}}.

Depending on the deployment of the cloud backbone, there are two options for corresponding GW information discovery and advertisement.

## Option 1: Link SID
Assume that SRv6 or SR-MPLS is running on the cloud backbone. SD-WAN tunnels will be established between the CPE and the corresponding GW. For each tunnel, a link SID will be allocated by the GW. Then the link SID will be notified from the GW to the CPE through a dedicated hello protocol or manual configuration.

A new Sub-TLV in the SD-WAN-Hybrid Tunnel Encapsulation attribute is defined as follows:

~~~
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Type=TBD1 (Link SID subTLV)  |  Length (2 Octets)            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   ~                            link SID                           ~
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #GW-info-link-sid}

## Option 2: Gateway ID + tunnel IP address

The GW site ID and the destination tunnel IP address can be used as the corresponding GW information. A new Sub-TLV in the SD-WAN-Hybrid Tunnel Encapsulation attribute is defined as follows:

~~~
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Type=TBD2 (GW info subTLV)   |  Length (2 Octets)            |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |      Egress GW Addr Family    | Address                       |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ (variable)                    +
   ~                                                               ~
   |           Egress GW Address                                   |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  SD-WAN Tunnel Addr Family    | Address                       |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ (variable)                    +
   ~                                                               ~
   |                SD-WAN Tunnel Address                          |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

# Exchange include/exclude transit list from destination edge to source edge

When a user tries to access certain service, the traffic may need to go through certain Cloud Availability Regions or Zones to get better security. Or the traffic can not go through certain Cloud Availability Regions or Zones due to the regulation. As described in {{I-D.draft-dmk-rtgwg-multisegment-sdwan}}, the traffic enforcement rule is indicated using the including/excluding transit list in the data plane which is encapsulated at the source edge.

The destination edge knows the traffic enforcement rules for each service behind it and need to pass the include/exclude transit list to the source edge. This document proposes to carry the list in the client route using Metadata Path Attribute defined in {{!I-D.draft-ietf-idr-5g-edge-service-metadata}}. Two new Sub-TLVs are defined to carry the include/exclude transit list.

# Security Considerations

This document does not introduce any new security considerations.

# IANA Considerations

TBD.

--- back

# Acknowledgements

The authors would like to thank Haibo Wang for his contribution to the document.
