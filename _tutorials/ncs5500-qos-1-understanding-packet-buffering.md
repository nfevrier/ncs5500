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

![06b-Pipelines.png]({{site.baseurl}}/images/06b-Pipelines.png){: .align-center}  

A big difference compared to our tradition routers based on run-to-completion architectures, is the fact we will only perform a single-lookup in ingress to figure out the destination of a received packet. So, this operation will happen in the ingress pipeline only.  

![05-SingleLookup.jpg]({{site.baseurl}}/images/05-SingleLookup.jpg){: .align-center}  

It will reduce the latency and globally will improve the performance.  
But it has significant implications on the packet buffering too. Indeed, the buffering in egress side will be minimal and in case of queue or port congestion, the packets will be stored in ingress. That also implies an advanced mechanism of request/authorization between arbitrers. That's what we will now describe.

__**Ingress-only buffering**__

Every packet received in the ingress pipeline will go through multiple stages. Each pipeline block has a specific role and a budget of resource access (read and write). The FWD block will be responsible for the packet destination lookup. That's where the routing decision is made and it will result in pointers to next-hop information and header encapsulation (among other things).  
Once the destination is identified, the packet is associated to a queue (a VOQ more precisely) but we will come back on this concept below.  
The ingress scheduler will contact its egress counterpart and will issue a packet transmission request: 

![07-CanIsend.jpg]({{site.baseurl}}/images/07-CanIsend.jpg){: .align-center}  

![08-SureShoot.jpg]({{site.baseurl}}/images/08-SureShoot.jpg){: .align-center}  

The egress scheduler will accept or refuse this request depending on the queue availability.  
If the queue is congested, the egress scheduler will not provide token for transmission and the packet will be stored in the ingress packet memories.  

![09-Congestion.jpg]({{site.baseurl}}/images/09-Congestion.jpg){: .align-center}  

Depending on the level of queue congestion, the packet will be stored inside the NPU or in the external DRAM.  Once the congestion state disappears, the egress scheduler will be able to issue the token for the most critical queue. The QoS configuration will dictate this behavior. If QoS is not configured, the First In First Out rule will apply.  

So the packets are buffered in different places in the process:  
- (ingress) On-chip Buffer: 16MB
- (ingress) external DRAM: 4BG
- (egress) Port Buffer: 16MB

![11-Buffers.jpg]({{site.baseurl}}/images/11-Buffers.jpg){: .align-center}  

For a given queue, packets are stored in OCB **or** DRAM.  
The decision to keep a queue in OCB or to evict it to the DRAM will depends on the numbers of packets buffered:  
- below 6000 packets or 200k Bytes, the queue will stay inside the chipset
- above one of these two thresholds, the packets are stored in DRAM

![12-OnChip.jpg]({{site.baseurl}}/images/12-OnChip.jpg){: .align-center}

![13-OffChip.jpg]({{site.baseurl}}/images/13-OffChip.jpg){: .align-center}

This management is done at the queue level only. It will not impact the other packets going to other destination or even to the same destination but in a different queue:

![14-QueueEviction.jpg]({{site.baseurl}}/images/14-QueueEviction.jpg){: .align-center}

As long as we have packets in this evicted queue, it will stay in the DRAM. Once emptied, the queue is moved back to internal buffer.  

On the egress side, we have a 16MB memory used to differentiate unicast and multicast traffic, and among them to separate high and low priority. That's the only level of classification available on the output buffer.  
In the future we will come back on the main difference of behavior between unicast and multicast, in a VOQ-only model.  

![15-Egress.jpg]({{site.baseurl}}/images/15-Egress.jpg){: .align-center}

__**Virtual Output Queues**__

Contrary traditional platforms in our SP portfolio (ASR9000, CRS, NCS6000, ...), the packets are not stored in the output queues in case of congestion.  
In the NCS5500 world, we are creating a virtual representation of the egress queues on the ingress side.  
Indeed, everytime a packet needs to be sent to a remote point but is not given the right to be transmitted to an egress pipeline, it will be stored in the ingress buffer (could it be inside the NPU or in the DRAM).  
That's the famous "Virtual Output Queues".  
For a given egress port or "attachment point", we have 8 queues in each ingress NPU.

In this example, the egress port HunGig0/7/0/1 is materialized by 8 queues in EACH ingress NPU

![16-VOQ1b.jpg]({{site.baseurl}}/images/16-VOQ1b.jpg){: .align-center}

![17-VOQ2b.jpg]({{site.baseurl}}/images/17-VOQ2b.jpg){: .align-center}

In the following show command, we take the perspective of the NPU 0 on Line Card 1, the queue is seen as remote.  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:R1#sh contr npu stats voq ingress interface hu 0/7/0/1 instance 0 location 0/1/CPU0

Interface Name    =    Hu0/7/0/1
Interface Handle  =      3800118
Asic Instance     =            0
VOQ Base          =         1328
Port Speed(kbps)  =    100000000
Local Port        =       <mark>remote</mark>
       ReceivedPkts    ReceivedBytes   DroppedPkts     DroppedBytes
-------------------------------------------------------------------
COS0 = 0               0               0               0
COS1 = 0               0               0               0
COS2 = 0               0               0               0
COS3 = 0               0               0               0
COS4 = 0               0               0               0
COS5 = 0               0               0               0
COS6 = 0               0               0               0
COS7 = 0               0               0               0
</code>
</pre>
</div>

But the queues also exist locally on the same NPU of the same line card, from an ingress pipeline perspective.  

![18-VOQ3b.jpg]({{site.baseurl}}/images/18-VOQ3b.jpg){: .align-center}

In the following output, we see that the queue is seen as local from the NPU 0 LC 0/7 perspective.  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:R1#sh contr npu stats voq ingress interface hu 0/7/0/1 instance 0 location 0/7/CPU0

Interface Name    =    Hu0/7/0/1
Interface Handle  =      3800118
Asic Instance     =            0
VOQ Base          =         1328
Port Speed(kbps)  =    100000000
Local Port        =        <mark>local</mark>
       ReceivedPkts    ReceivedBytes   DroppedPkts     DroppedBytes
-------------------------------------------------------------------
COS0 = 0               0               0               0
COS1 = 9               1107            0               0
COS2 = 0               0               0               0
COS3 = 0               0               0               0
COS4 = 0               0               0               0
COS5 = 0               0               0               0
COS6 = 0               0               0               0
COS7 = 139289          28863790        0               0
RP/0/RP0/CPU0:TME-5508-1-6.5.1#
</code>
</pre>
</div>



![18-VOQ3.jpg]({{site.baseurl}}/images/18-VOQ3.jpg){: .align-center}



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
