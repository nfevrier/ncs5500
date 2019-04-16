---
published: false
date: '2019-04-12 17:00 +0200'
title: NCS5500 QoS Part 1 - Understanding Packet Buffering
author: Nicolas Fevrier
excerpt: 'First part of the NCS5500 QoS Series: Packet Buffering'
---
{% include toc icon="table" title="NCS5500 Buffering Architecture" %} 

You can find more content related to NCS5500 including routing memory management, VRF, URPF, ACLs, Netflow following this [link](https://xrdocs.io/ncs5500/tutorials/).

## Cisco NCS5500 QOS Series - 01 - Introduction to Packet BUFFERING

This first blog post will help understanding key concepts on the buffering architecture and clarify things like VOQ-only, single-lookup and ingress-buffering only designs.  
It's necessary to detail all these mechanisms to understand later the subtleties of the QoS implementation, including the scales and limitations.  

### Video

The short version of this article is available in the following Youtube video:

[https://www.youtube.com/watch?v=_ozPYN6Ej9Y](https://www.youtube.com/watch?v=_ozPYN6Ej9Y)

<iframe src="https://www.youtube.com/watch?v=_ozPYN6Ej9Y" width="800" height="550" frameborder="0" allowfullscreen webkitallowfullscreen msallowfullscreen></iframe>

### DNX ASICs in NCS5500 family

All the NCS5500 routers are powered by the Broadcom StrataDNX (or DNX) Network Processor Units (NPUs).  
These ASICs can be used in standalone mode (Systems on Chip) or interconnected through one or multiple Fabric Engines.  

![01-dnx-fe3600.jpg]({{site.baseurl}}/images/01-dnx-fe3600.jpg){: .align-center}  
![02-dnx-soc.jpg]({{site.baseurl}}/images/02-dnx-soc.jpg){: .align-center}  

All Jericho and Qumran options are following a very similar architecture, they are using resources inside the ASIC (packet memory, routing information memories, next-hop memories, encapsulation memories, statistics memories, etc etc) and also external to the chipset (like the DRAM used for packet buffering).  
Some systems are equipped with external TCAMs or not, that's how we differentiate base and scale systems/LCs.  
On the Jericho side:

![03-Jericho x3.jpg]({{site.baseurl}}/images/03-Jericho x3.jpg){: .align-center}  

- Jericho ASICs are used in NCS5502, NCS5502-SE, and in following line cards: NC55-36X100G, NC55-24H12F-SE, NC55-24X100G-SE, NC55-18H18F, NC55-6X2H-DWDM, NC55-36X100G-S
- Jericho+ (with "normal LPM") ASICs are used in NCS55A1-36H-S, NCS55A1-36H-SE-s, all NCS55A2-MOD-xx and in following line cards: NC55-36X100G-A-SE, NC55-MOD-A-S, NC55-MOD-A-SE-S
- Jericho+ with large LPM is used in NCS-55A1-24H

On the Qumran side:

![04-Qumran x3.jpg]({{site.baseurl}}/images/04-Qumran x3.jpg){: .align-center}  

- Qumran-MX is used in NCS5501 and NCS5501-SE
- Qumran-MX with 2nd generation eTCAM (OP) is used in the scale version of RSP4 / NCS560
- Qumran-AX is used in the N540-24Z8Q2C-SYS

### DNX ASICs architecture

All the DNX forwarding ASICs used in the NCS550 portfolio are made of two cores (actually, only the Qumran-AX used in NCS540 is single core), and they are all made of an ingress and an egress pipeline.

![06-Pipelines.jpg]({{site.baseurl}}/images/06-Pipelines.jpg){: .align-center}  

A packet processed in the forwarding ASIC will always go through the ingress pipeline and the egress pipeline, could it be in the same NPU or in different NPUs (connected through a fabric engine).  

A big difference compared to our tradition routers based on run-to-completion architectures, is the fact we will only perform a single-lookup in ingress to figure out the destination of a received packet. So, this operation will happen in the ingress pipeline only.  

![05-SingleLookup.jpg]({{site.baseurl}}/images/05-SingleLookup.jpg){: .align-center}  

It will reduce the latency and globally will improve the performance.  
But it has significant implications on the packet buffering too. Indeed, the buffering in egress side will be minimal and in case of queue or port congestion, the packets will be stored in ingress. That's what we will now describe.



<div class="highlighter-rouge">
<pre class="highlight">
<code>
CODE CODE CODE
<mark>NC55-36X100G</mark> 
CODE CODE CODE
</code>
</pre>
</div>

Notice-blablabla
{: .notice--info}
