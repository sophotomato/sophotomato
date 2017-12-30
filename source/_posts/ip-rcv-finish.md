---
title: ip_rcv_finish()处理sk_buff数据分析
date: 2017-10-10 21:40:12
tags:
- 内核网络
- kernel
---

> 所有源码来自[kernel.org](https://www.kernel.org)
> 内核版本号3.10

```c {.line-numbers}
static int ip_rcv_finish(struct sk_buff *skb)
{
	const struct iphdr *iph = ip_hdr(skb);
	struct rtable *rt;

	if (sysctl_ip_early_demux && !skb_dst(skb)) {
		const struct net_protocol *ipprot;
		int protocol = iph->protocol;

		ipprot = rcu_dereference(inet_protos[protocol]);
		if (ipprot && ipprot->early_demux) {
			ipprot->early_demux(skb);
			/* must reload iph, skb->head might have changed */
			iph = ip_hdr(skb);
		}
	}

	/*
	 *	Initialise the virtual path cache for the packet. It describes
	 *	how the packet travels inside Linux networking.
	 */
	if (!skb_dst(skb)) {	//方法skb_dst()用于检查是否有于skb相关联的dst对象
		int err = ip_route_input_noref(skb, iph->daddr, iph->saddr,
					       iph->tos, skb->dev);
		if (unlikely(err)) {
			if (err == -EXDEV)
				NET_INC_STATS_BH(dev_net(skb->dev),
						 LINUX_MIB_IPRPFILTER);
			goto drop;
		}
	}

#ifdef CONFIG_IP_ROUTE_CLASSID
	if (unlikely(skb_dst(skb)->tclassid)) {
		struct ip_rt_acct *st = this_cpu_ptr(ip_rt_acct);
		u32 idx = skb_dst(skb)->tclassid;
		st[idx&0xFF].o_packets++;
		st[idx&0xFF].o_bytes += skb->len;
		st[(idx>>16)&0xFF].i_packets++;
		st[(idx>>16)&0xFF].i_bytes += skb->len;
	}
#endif

	if (iph->ihl > 5 && ip_rcv_options(skb))
		goto drop;

	rt = skb_rtable(skb);	//获取rtable，根据rt_type，统计数据？
	if (rt->rt_type == RTN_MULTICAST) {
		IP_UPD_PO_STATS_BH(dev_net(rt->dst.dev), IPSTATS_MIB_INMCAST,
				skb->len);
	} else if (rt->rt_type == RTN_BROADCAST)
		IP_UPD_PO_STATS_BH(dev_net(rt->dst.dev), IPSTATS_MIB_INBCAST,
				skb->len);

	return dst_input(skb);

drop:
	kfree_skb(skb);
	return NET_RX_DROP;
}

/*************************************
 *
 *************************************/

```
* 获取ip报文头；
* 检测skb是否有关联的dst_entry对象，若没有则，使用ip_route_input_noref()进行查找；
* 如果ip报文头大于20个字节((iphdr)->ihl)， 则调用方法ip_rcv_options()对可选项部分进行处理；
* 通过调用dst_input()方法，来执行(dst_entry)->input()方法对skb进行处理，回调函数input()是在路由选择子系统中查找时会设置，如需要转发数据包，则将其设置为ip_forward()，数据包的目的地址为本机，则将其设置为ip_local_deliver()，对于组播数据包，可能将其设置为ip_mr_input()。
