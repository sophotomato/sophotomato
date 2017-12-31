---
title: IPv4路由子选择系统
date: 2017-10-09 19:25:01
tags:
- 内核网络
- kernel
---

> 所有源码来自[kernel.org](https://www.kernel.org)
> 内核版本号3.10

#### IPv4处理接收包过程
```c
ip_rcv()
  ->ip_rcv_finish()
    ->ip_route_input_noref()
      ->ip_route_input_slow()
        ->fib_lookup()
          ->fib_table_lookup()
```
<!-- more -->
> 关于`fib_table_lookup()`查表的内容可查看本站其他博文。
##### 路由表

```c {.line-numbers}
/*************************************
 * (struct rtable) = (struct dst_entry) = (skb->_skb_refdst)
 */
struct rtable {
	struct dst_entry	dst;

	int			rt_genid;
	unsigned int		rt_flags;
	__u16			rt_type;
	__u8			rt_is_input;  //输入路由设置为1
	__u8			rt_uses_gateway; //下一跳为网关设置为1

	int			rt_iif;

	/* Info on neighbour */
	__be32			rt_gateway;

	/* Miscellaneous cached information */
	u32			rt_pmtu; //路径MTU(路途上最小的MTU)

	struct list_head	rt_uncached;
};
/*********************************
->rt_flags
  RTCF_BROADCAST:目标地址为广播地址。在__mkroute_output()和ip_route_input_slow()中设置
  RTCF_MULTICAST:目标地址为组播地址。在ip_route_input_mc()和__mkroute_output()设置。
  RTCF_DOREDIRECT:必需发送一条ICMPv4重定向消息。在__mkroute_input()中设置。
  RTCF_LOCAL:目的地为当前主机。在ip_route_input_slow()、__mkroute_output()、ip_route_input_mc()、__ip_route_output_key()中设置。
*/
```

```c {.line-numbers}
struct fib_result {
	unsigned char	prefixlen;	//前缀长度，表示子网掩码
	unsigned char	nh_sel;		//下一跳数量
	unsigned char	type;		//决定处理数据包的方式
	unsigned char	scope;
	u32		tclassid;
	struct fib_info *fi;		//fib_info指向下一跳的引用(fib_nh)
	struct fib_table *table;	//指向用于查找的table
	struct list_head *fa_head;	//指向一个fib_alias列表
};

```
##### FIB表
```c {.line-numbers}
/*************************************
 * struct trie *t = (struct trie *) (struct fib_table)->tb_data;
 * struct rt_trie_node *n = t->trie;
 *
 * (struct leaf *) = (struct leaf *) n;
 * (struct tnode *) = (struct tnode *) n;
 *************************************/
struct fib_table {
	struct hlist_node	tb_hlist;
	u32			tb_id;	//路由选择表标识符
	int			tb_default;
	int			tb_num_default;	//表中包含默认路由数
	unsigned long		tb_data[0];	//路由选择条目对象(trie)的占位符。
};
```
```c {.line-numbers}
/*
 * This structure contains data shared by many of routes.
 */

struct fib_info {
	struct hlist_node	fib_hash;
	struct hlist_node	fib_lhash;
	struct net		*fib_net;  //所属network namespace
	int			fib_treeref; //包含该fib_info的fib_alias对象的个数
	atomic_t		fib_clntref;
	unsigned int		fib_flags;
	unsigned char		fib_dead;
	unsigned char		fib_protocol; //段后注释
	unsigned char		fib_scope;
	unsigned char		fib_type;
	__be32			fib_prefsrc;
	u32			fib_priority;  //路由优先级，默认0（最大优先级）
	u32			*fib_metrics;
#define fib_mtu fib_metrics[RTAX_MTU-1]
#define fib_window fib_metrics[RTAX_WINDOW-1]
#define fib_rtt fib_metrics[RTAX_RTT-1]
#define fib_advmss fib_metrics[RTAX_ADVMSS-1]
	int			fib_nhs;   //下一跳的数量
#ifdef CONFIG_IP_ROUTE_MULTIPATH
	int			fib_power;
#endif
	struct rcu_head		rcu;
	struct fib_nh		fib_nh[0];
#define fib_dev		fib_nh[0].nh_dev
};
/**********************************
->fib_protocol
RTPROT_UNSPEC:一个错误值
RTPROT_REDIRECT:路由选择条目是因收到ICMP重定向消息而创建的。仅用于IPv6中
RTPROT_KERNEL:路由选择条目由内核创建
RTPROT_BOOT:路由是由管理员添加，切且没有使用proto static 修饰符
RTPROT_STATIC:路由是由管理员添加
RTPROT_RA:RDISC/ND路由器通告，仅用于IPv6子系统
************************************/
```
```c {.line-numbers}
struct fib_alias {
	struct list_head	fa_list;
	struct fib_info		*fa_info;
	u8			fa_tos;
	u8			fa_type;
	u8			fa_state;
	struct rcu_head		rcu;
};
```

```c {.line-numbers}
/* 路由类型 */
enum {
	RTN_UNSPEC,
	RTN_UNICAST,		/* Gateway or direct route	*/
	RTN_LOCAL,		/* Accept locally		*/
	RTN_BROADCAST,		/* Accept locally as broadcast,
				   send as broadcast */
	RTN_ANYCAST,		/* Accept locally as broadcast,
				   but send as unicast */
	RTN_MULTICAST,		/* Multicast route		*/
	RTN_BLACKHOLE,		/* Drop				*/
	RTN_UNREACHABLE,	/* Destination is unreachable   */
	RTN_PROHIBIT,		/* Administratively prohibited	*/
	RTN_THROW,		/* Not in this table		*/
	RTN_NAT,		/* Translate this address	*/
	RTN_XRESOLVE,		/* Use external resolver	*/
	__RTN_MAX
};

```
Linux符号 | 错误值 | 范围
---- | ---- | ----
RTN_UNSPEC | 0 | RT_SCOPE_NOWHERE
RTN_UNICAST | 0 | RT_SCOPE_UNIVERSE
RTN_LOCAL | 0 | RT_SCOPE_HOST
RTN_BROADCAST | 0 | RT_SCOPE_LINK
RTN_ANYCAST | 0 | RT_SCOPE_LINK
RTN_MULTICAST | 0 | RT_SCOPE_UNIVERSE
RTN_BLACKHOLE | -EINVAL | RT_SCOPE_UNIVERSE
RTN_UNREACHABLE | -EHOSTUNREACH | RT_SCOPE_UNIVERSE
RTN_PROHIBIT | -EACCES | RT_SCOPE_UNIVERSE
RTN_THROW | -EAGAIN | RT_SCOPE_UNIVERSE
RTN_NAT | -EINVAL | RT_SCOPE_NOWHERE
RTN_XRESOLVE | -EINVAL | RT_SCOPE_NOWHERE

##### 下一跳
```c {.line-numbers}
struct fib_nh {
	struct net_device	*nh_dev; //将流量传输到下一跳所使用的网络设备----网络设备被禁用，将发送NETDEV_DOWN通知。处理这种事件的FIB回调函数为fib_netdev_event()
	struct hlist_node	nh_hash;
	struct fib_info		*nh_parent;
	unsigned int		nh_flags;
	unsigned char		nh_scope;
#ifdef CONFIG_IP_ROUTE_MULTIPATH
	int			nh_weight;
	int			nh_power;
#endif
#ifdef CONFIG_IP_ROUTE_CLASSID
	__u32			nh_tclassid;
#endif
	int			nh_oif;
	__be32			nh_gw;
	__be32			nh_saddr;
	int			nh_saddr_genid;
	struct rtable __rcu * __percpu *nh_pcpu_rth_output;
	struct rtable __rcu	*nh_rth_input;
	struct fnhe_hash_bucket	*nh_exceptions;
};
```
###### 缓存下一跳
```c {.line-numbers}
/************************
 * 该函数完成下一跳的缓存
 ************************/
static bool rt_cache_route(struct fib_nh *nh, struct rtable *rt)
{
	struct rtable *orig, *prev, **p;
	bool ret = true;

	if (rt_is_input_route(rt)) {
		p = (struct rtable **)&nh->nh_rth_input;
	} else {
		p = (struct rtable **)__this_cpu_ptr(nh->nh_pcpu_rth_output);
	}
	orig = *p;

	prev = cmpxchg(p, orig, rt);
	if (prev == orig) {
		if (orig)
			rt_free(orig);
	} else
		ret = false;

	return ret;
}
```
##### 下一跳例外(nexthop exception)
```c {.line-numbers}
struct fib_nh_exception {
	struct fib_nh_exception __rcu	*fnhe_next;
	__be32				fnhe_daddr;
	u32				fnhe_pmtu;
	__be32				fnhe_gw;
	unsigned long			fnhe_expires;
	struct rtable __rcu		*fnhe_rth;
	unsigned long			fnhe_stamp;
};
/*************************************
fib_nh_exception对象是由方法update_or_create_fnhe()创建的
*************************************/
```
#### 策略路由选择
不使用策略路由选择（没有设置CONFIG_IP_MULTIPLE_TABLES时），将创建两个路由选择表：本地表和主表。主表ID为254（RT_TABLE_MAIN）,本地表ID为255（RT_TABLE_LOCAL）
##### fib_info的创建
```c {.line-numbers}
struct fib_info *fib_create_info(struct fib_config *cfg)
{
	int err;
	struct fib_info *fi = NULL;
	struct fib_info *ofi;
	int nhs = 1;
	struct net *net = cfg->fc_nlinfo.nl_net;

	if (cfg->fc_type > RTN_MAX)
		goto err_inval;

	/* Fast check to catch the most weird cases */
	if (fib_props[cfg->fc_type].scope > cfg->fc_scope)
		goto err_inval;

#ifdef CONFIG_IP_ROUTE_MULTIPATH
	if (cfg->fc_mp) {
		nhs = fib_count_nexthops(cfg->fc_mp, cfg->fc_mp_len);
		if (nhs == 0)
			goto err_inval;
	}
#endif

	err = -ENOBUFS;
	if (fib_info_cnt >= fib_info_hash_size) {
		unsigned int new_size = fib_info_hash_size << 1;
		struct hlist_head *new_info_hash;
		struct hlist_head *new_laddrhash;
		unsigned int bytes;

		if (!new_size)
			new_size = 16;
		bytes = new_size * sizeof(struct hlist_head *);
		new_info_hash = fib_info_hash_alloc(bytes);
		new_laddrhash = fib_info_hash_alloc(bytes);
		if (!new_info_hash || !new_laddrhash) {
			fib_info_hash_free(new_info_hash, bytes);
			fib_info_hash_free(new_laddrhash, bytes);
		} else
			fib_info_hash_move(new_info_hash, new_laddrhash, new_size);

		if (!fib_info_hash_size)
			goto failure;
	}

	fi = kzalloc(sizeof(*fi)+nhs*sizeof(struct fib_nh), GFP_KERNEL);
	if (fi == NULL)
		goto failure;
	if (cfg->fc_mx) {
		fi->fib_metrics = kzalloc(sizeof(u32) * RTAX_MAX, GFP_KERNEL);
		if (!fi->fib_metrics)
			goto failure;
	} else
		fi->fib_metrics = (u32 *) dst_default_metrics;
	fib_info_cnt++;

	fi->fib_net = hold_net(net);
	fi->fib_protocol = cfg->fc_protocol;
	fi->fib_scope = cfg->fc_scope;
	fi->fib_flags = cfg->fc_flags;
	fi->fib_priority = cfg->fc_priority;
	fi->fib_prefsrc = cfg->fc_prefsrc;
	fi->fib_type = cfg->fc_type;

	fi->fib_nhs = nhs;
	change_nexthops(fi) {
		nexthop_nh->nh_parent = fi;
		nexthop_nh->nh_pcpu_rth_output = alloc_percpu(struct rtable __rcu *);
		if (!nexthop_nh->nh_pcpu_rth_output)
			goto failure;
	} endfor_nexthops(fi)

	if (cfg->fc_mx) {
		struct nlattr *nla;
		int remaining;

		nla_for_each_attr(nla, cfg->fc_mx, cfg->fc_mx_len, remaining) {
			int type = nla_type(nla);

			if (type) {
				u32 val;

				if (type > RTAX_MAX)
					goto err_inval;
				val = nla_get_u32(nla);
				if (type == RTAX_ADVMSS && val > 65535 - 40)
					val = 65535 - 40;
				if (type == RTAX_MTU && val > 65535 - 15)
					val = 65535 - 15;
				fi->fib_metrics[type - 1] = val;
			}
		}
	}

	if (cfg->fc_mp) {
#ifdef CONFIG_IP_ROUTE_MULTIPATH
		err = fib_get_nhs(fi, cfg->fc_mp, cfg->fc_mp_len, cfg);
		if (err != 0)
			goto failure;
		if (cfg->fc_oif && fi->fib_nh->nh_oif != cfg->fc_oif)
			goto err_inval;
		if (cfg->fc_gw && fi->fib_nh->nh_gw != cfg->fc_gw)
			goto err_inval;
#ifdef CONFIG_IP_ROUTE_CLASSID
		if (cfg->fc_flow && fi->fib_nh->nh_tclassid != cfg->fc_flow)
			goto err_inval;
#endif
#else
		goto err_inval;
#endif
	} else {
		struct fib_nh *nh = fi->fib_nh;

		nh->nh_oif = cfg->fc_oif;
		nh->nh_gw = cfg->fc_gw;
		nh->nh_flags = cfg->fc_flags;
#ifdef CONFIG_IP_ROUTE_CLASSID
		nh->nh_tclassid = cfg->fc_flow;
		if (nh->nh_tclassid)
			fi->fib_net->ipv4.fib_num_tclassid_users++;
#endif
#ifdef CONFIG_IP_ROUTE_MULTIPATH
		nh->nh_weight = 1;
#endif
	}

	if (fib_props[cfg->fc_type].error) {
		if (cfg->fc_gw || cfg->fc_oif || cfg->fc_mp)
			goto err_inval;
		goto link_it;
	} else {
		switch (cfg->fc_type) {
		case RTN_UNICAST:
		case RTN_LOCAL:
		case RTN_BROADCAST:
		case RTN_ANYCAST:
		case RTN_MULTICAST:
			break;
		default:
			goto err_inval;
		}
	}

	if (cfg->fc_scope > RT_SCOPE_HOST)
		goto err_inval;

	if (cfg->fc_scope == RT_SCOPE_HOST) {
		struct fib_nh *nh = fi->fib_nh;

		/* Local address is added. */
		if (nhs != 1 || nh->nh_gw)
			goto err_inval;
		nh->nh_scope = RT_SCOPE_NOWHERE;
		nh->nh_dev = dev_get_by_index(net, fi->fib_nh->nh_oif);
		err = -ENODEV;
		if (nh->nh_dev == NULL)
			goto failure;
	} else {
		change_nexthops(fi) {
			err = fib_check_nh(cfg, fi, nexthop_nh);
			if (err != 0)
				goto failure;
		} endfor_nexthops(fi)
	}

	if (fi->fib_prefsrc) {
		if (cfg->fc_type != RTN_LOCAL || !cfg->fc_dst ||
		    fi->fib_prefsrc != cfg->fc_dst)
			if (inet_addr_type(net, fi->fib_prefsrc) != RTN_LOCAL)
				goto err_inval;
	}

	change_nexthops(fi) {
		fib_info_update_nh_saddr(net, nexthop_nh);
	} endfor_nexthops(fi)

link_it:
	ofi = fib_find_info(fi); //找到类似的对象，新创建的fib_info释放，将找到对象的引用计数（fib_treeref）加1
	if (ofi) {
		fi->fib_dead = 1;
		free_fib_info(fi);
		ofi->fib_treeref++;
		return ofi;
	}

	fi->fib_treeref++;
	atomic_inc(&fi->fib_clntref);
	spin_lock_bh(&fib_info_lock);
	hlist_add_head(&fi->fib_hash,
		       &fib_info_hash[fib_info_hashfn(fi)]);
	if (fi->fib_prefsrc) {
		struct hlist_head *head;

		head = &fib_info_laddrhash[fib_laddr_hashfn(fi->fib_prefsrc)];
		hlist_add_head(&fi->fib_lhash, head);
	}
	change_nexthops(fi) {
		struct hlist_head *head;
		unsigned int hash;

		if (!nexthop_nh->nh_dev)
			continue;
		hash = fib_devindex_hashfn(nexthop_nh->nh_dev->ifindex);
		head = &fib_info_devhash[hash];
		hlist_add_head(&nexthop_nh->nh_hash, head);
	} endfor_nexthops(fi)
	spin_unlock_bh(&fib_info_lock);
	return fi;

err_inval:
	err = -EINVAL;

failure:
	if (fi) {
		fi->fib_dead = 1;
		free_fib_info(fi);
	}

	return ERR_PTR(err);
}
```
#### 生成ICMPv4重定向消息
* `__mkroute_input()`方法中，必要时会设置`RTCF_DOREDIRECT`
* `ip_forward()`中，调用方法`ip_rt_send_redirect()`来发送ICMPv4重定向消息

满足如下条件会设置`RTCF_DOREDIRECT`

* 输入设备和输出设备相同
* procfs条目/proc/sys/net/ipv4/conf/<dev name>/send_redirects被设置
* 输出设备为共享介质，或原地址和下一跳的网关同属一个子网
```c {.line-numbers}
if (out_dev == in_dev && err && IN_DEV_TX_REDIRECTS(out_dev) &&
	    (IN_DEV_SHARED_MEDIA(out_dev) ||
	     inet_addr_onlink(out_dev, saddr, FIB_RES_GW(*res)))) {
		flags |= RTCF_DOREDIRECT;
		do_cache = false;
	}
```
```c {.line-numbers}
/*
 *	We now generate an ICMP HOST REDIRECT giving the route
 *	we calculated.
 */	//检查是否设置RTCF_DOREDIRECT，是否不包含IP选项（严格路由），且不是IPsec数据包
  if (rt->rt_flags&RTCF_DOREDIRECT && !opt->srr && !skb_sec_path(skb))
    ip_rt_send_redirect(skb);
void ip_rt_send_redirect(struct sk_buff *skb)
{
  .................
  icmp_send(skb, ICMP_REDIRECT, ICMP_REDIR_HOST,
    rt_nexthop(rt, ip_hdr(skb)->daddr));
  ....................
}
```
#### 接收ICMPv4重定向消息
`__ip_do_redirect`完成接收。
```c {.line-numbers}
static void __ip_do_redirect(struct rtable *rt, struct sk_buff *skb, struct flowi4 *fl4,
			     bool kill_route)
{
	__be32 new_gw = icmp_hdr(skb)->un.gateway;
	__be32 old_gw = ip_hdr(skb)->saddr;
	struct net_device *dev = skb->dev;
	struct in_device *in_dev;
	struct fib_result res;
	struct neighbour *n;
	struct net *net;

	switch (icmp_hdr(skb)->code & 7) {
	case ICMP_REDIR_NET:
	case ICMP_REDIR_NETTOS:
	case ICMP_REDIR_HOST:
	case ICMP_REDIR_HOSTTOS:
		break;

	default:
		return;
	}

	if (rt->rt_gateway != old_gw)
		return;

	in_dev = __in_dev_get_rcu(dev);
	if (!in_dev)
		return;

	net = dev_net(dev);
	if (new_gw == old_gw || !IN_DEV_RX_REDIRECTS(in_dev) ||
	    ipv4_is_multicast(new_gw) || ipv4_is_lbcast(new_gw) ||
	    ipv4_is_zeronet(new_gw))
		goto reject_redirect;

	if (!IN_DEV_SHARED_MEDIA(in_dev)) {
		if (!inet_addr_onlink(in_dev, new_gw, old_gw))
			goto reject_redirect;
		if (IN_DEV_SEC_REDIRECTS(in_dev) && ip_fib_check_default(new_gw, dev))
			goto reject_redirect;
	} else {
		if (inet_addr_type(net, new_gw) != RTN_UNICAST)
			goto reject_redirect;
	}
  //在邻接子系统中查找
	n = ipv4_neigh_lookup(&rt->dst, NULL, &new_gw);
	if (n) {
		if (!(n->nud_state & NUD_VALID)) {
			neigh_event_send(n, NULL);
		} else {
			if (fib_lookup(net, fl4, &res) == 0) {
				struct fib_nh *nh = &FIB_RES_NH(res);
        //创建或更新fib nexthop exception
				update_or_create_fnhe(nh, fl4->daddr, new_gw,
						      0, 0);
			}
			if (kill_route)
				rt->dst.obsolete = DST_OBSOLETE_KILL;
			call_netevent_notifiers(NETEVENT_NEIGH_UPDATE, n);
		}
		neigh_release(n);
	}
	return;

reject_redirect:
#ifdef CONFIG_IP_ROUTE_VERBOSE
	if (IN_DEV_LOG_MARTIANS(in_dev)) {
		const struct iphdr *iph = (const struct iphdr *) skb->data;
		__be32 daddr = iph->daddr;
		__be32 saddr = iph->saddr;

		net_info_ratelimited("Redirect from %pI4 on %s about %pI4 ignored\n"
				     "  Advised path = %pI4 -> %pI4\n",
				     &old_gw, dev->name, &new_gw,
				     &saddr, &daddr);
	}
#endif
	;
}
```
