---
title: Selective Synchronization for RPKI to Router Protocol
abbrev: Selective Synchronization for RTR
docname: draft-geng-sidrops-rtr-selective-sync-00
obsoletes:
updates:
date:
category: std
submissionType: IETF

ipr: trust200902
area: ops
workgroup: sidrops
keyword: Internet-Draft

author:
 -
  ins: N. Geng
  name: Nan Geng
  organization: Huawei
  email: gengnan@huawei.com
  city: Beijing
  country: China
 -
  ins: S. Zhuang
  name: Shunwan Zhuang
  organization: Huawei
  email: zhuangshunwan@huawei.com
  city: Beijing
  country: China
 -
  ins: M. Huang
  name: Mingqing Huang
  organization: Huawei
  email: huangmingqing@huawei.com
  city: Beijing
  country: China

normative:
  RFC6810:
  RFC8210:
  I-D.ietf-sidrops-8210bis:

informative:
  I-D.van-beijnum-sidrops-pathrpki:
  I-D.ietf-grow-rpki-as-cones:
  I-D.spaghetti-sidrops-rpki-asgroup:
...

--- abstract
The RPKI-to-Router (RTR) protocol synchronizes all the verified RPKI data to the router. This document proposes to extend the existing RTR protocol to support selective data synchronization. Selective synchronization can avoid some unnecessary synchronization overhead. The router can obtain only the data that it needs, and does not need to save the data that it does not need. 

--- middle

# Introduction {#sec-intro}
The RPKI-to-Router (RTR) protocol is a simple but reliable approach, which help synchronize the validated RPKI data from a trusted cache to routers. There are already several versions of the protocol {{!RFC6810}}{{!RFC8210}}{{!I-D.ietf-sidrops-8210bis}}. The supported types of data that can be transferred increase, which is shown in {{tab-version}}. 

| Version 0 | Version 1 | Version 2 |
|:------:+:------:+:------:|
| IPv4 Prefix | IPv4 Prefix | IPv4 Prefix |
| IPv6 Prefix | IPv6 Prefix | IPv6 Prefix |
|             | Router Key  | Router Key  |
|             |             | ASPA        |
{: #tab-version title="Supported data types in different versions of the RTR protocol"}

The RTR protocol keeps the synchronization of all types of data, and selective synchronization is not supported. However, routers may be interested in a part of data types, instead of all. In such cases, storing unused data on the router is unreasonable, and synchronizing all types of data will induce some unnecessary transmission and storage overhead. Since multiple types of data are transmitted together, the router cannot use any type of these data unless it waits for all data to complete transmission. Furthermore, there may be more types of data in the cache {{?I-D.van-beijnum-sidrops-pathrpki}}{{?I-D.ietf-grow-rpki-as-cones}}{{?I-D.spaghetti-sidrops-rpki-asgroup}}, which makes the above issue worse. 

This document describes the synchronization problem of the RTR protocol and provides some possible solutions. 

## Requirements Language

{::boilerplate bcp14-tagged}

# Problem Statement {#sec-ps}
The RTR protocol does not distinguish data types in the cache. Different types of data share one serial number and one End of Data PDU. When the Relay Party (RP) synchronizes the cache to the router, various PDUs, such as IPv4 Prefix, IPv6 Prefix, Router Key, and ASPA, are mixed. The router cannot select one or more really required PDUs or deny receiving a certain kind of PDU. For example, if the router does not support or enable ASPA. The ASPA PDU messages will still be transmitted. The required Data PDU type cannot be flexibly selected by the router.

The negative effects of the above problem are as follows: 

- Storing unused data on the router, which is unreasonable. 

- Unnecessary transmission and storage overhead. 

- Inefficient end-of-transmission acknowledgment. Multiple types of data are transmitted together. The router cannot use any type of these data unless it waits for all data to complete transmission. 

The above negative effects will become worse when there are more kinds of RPKI data available {{?I-D.van-beijnum-sidrops-pathrpki}}{{?I-D.ietf-grow-rpki-as-cones}}{{?I-D.spaghetti-sidrops-rpki-asgroup}}. 
The main problem of the RTR protocol is the lack of selective synchronization capability. 

Another problem of the protocol is the low extension capability. When there are new PDUs defined for transmission, a new RTR version need to be issued. The new version protocol is not well compatible with the older ones, which induces some challenges on implementation and deployment. This document will define some new PDUs that may not necessarily require a new protocol version. 

# Preliminary Solutions {#sec-solution}
This section preliminarily proposes some independent solutions for achiving selective synchronization in the RTR protocol, while trying to keep the protocol's simplicity. 

## Subscribing Data PDU
Define a new type of PDU called Subscribing Data PDU. The new PDU will indicate the data types that the router is interested in. An example format of the PDU is shown in {{fig-subscribe}}. The field of PDU type is TBD. The Data Type fields indicate the interested data types (i.e., 4: IPv4 Prefix, 6: IPv6 Prefix, 9: Router Key, 11: ASPA). 

The router can send the Subscribing Data PDU to the cache. After finishing the subscribing, the following PDUs, including Serial Notify, Serial Query, Reset Query, Cache Response, and Cache Reset, are only for the subscribed data. If the router wants to modify the subscription, a new Subscribing Data PDU can be sent for overwriting the previous subscription. 

~~~
0          8          16         24        31
.-------------------------------------------.
| Protocol |   PDU    |                     |
| Version  |   Type   |         zero        |
|          |          |                     |
+-------------------------------------------+
|                                           |
|                  Length                   |
|                                           |
+-------------------------------------------+
|  Data    |          |         | Data      |
|  Type 1  | ...      | ...     | Type N    |
|          |          |         |           |
`-------------------------------------------'
~~~
{: #fig-subscribe  title="An example format of Subscribing Data PDU"}

## PDUs with Data Type Field
The existing PDUs, including Serial Notify, Serial Query, Reset Query, Cache Response, and Cache Reset, can be extended to carry the Data Type field. The values of the Data Type field can be 4 for IPv4 Prefix, for IPv6 Prefix, 9 for Router Key, and 11 for ASPA. An example format of the extended Serial Query PDU is shown in {{fig-serial}}. A router can send the extended Serial Query PDU for requesting a specific type of data. 

~~~  
0          8          16         24        31
.-------------------------------------------.
| Protocol |   PDU    |                     |
| Version  |   Type   |     Session ID      |
|    2     |    1     |                     |
+-------------------------------------------+
|                                           |
|                 Length=16                 |
|                                           |
+-------------------------------------------+
|                                           |
|                  Data Type                |
|                                           |
+-------------------------------------------+
|                                           |
|               Serial Number               |
|                                           |
`-------------------------------------------'
~~~
{: #fig-serial  title="An example format of extended Serial Query PDU"}

## End of Specific Data PDU
End of Data PDU tells the router that all the requested data are synchronized. The End of Specific Data PDU can be defined for indicating a specific type of data has been synchronized. An example format of End of Specific Data PDU is shown in {{fig-end-pdu}}. The field of PDU type is TBD. The Data Type field indicate the interested data types (i.e., 4: IPv4 Prefix, 6: IPv6 Prefix, 9: Router Key, 11: ASPA). 

The type of data specified in End of Specific Data PDU will become ready for use. The router does not need to wait for all the data to complete transmission before it can use the specified data. 

~~~
0          8          16         24        31
.-------------------------------------------.
| Protocol |   PDU    |                     |
| Version  |   Type   |     Session ID      |
|          |          |                     |
+-------------------------------------------+
|                                           |
|                 Length=24                 |
|                                           |
+-------------------------------------------+
|                                           |
|               Serial Number               |
|                                           |
+-------------------------------------------+
|                                           |
|              Refresh Interval             |
|                                           |
+-------------------------------------------+
|                                           |
|               Retry Interval              |
|                                           |
+-------------------------------------------+
|                                           |
|              Expire Interval              |
|                                           |
+-------------------------------------------+
|                                           |
|                 Data Type                 |
|                                           |
`-------------------------------------------'
~~~
{: #fig-end-pdu  title="An example format of End of Specific Data PDU"}

# Security Considerations {#sec-security}

TBD

# IANA Considerations {#sec-iana}

TBD

--- back

# Acknowledgements
{:numbered="false"}

TBD





