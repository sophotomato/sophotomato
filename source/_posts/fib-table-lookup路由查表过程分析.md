---
title: fib_table_lookup路由查表过程分析 - Linux-3.10
date: 2017-09-29 22:34:12
tags:
- 内核网络
- kernel
---
> 本文主要分析fib_table_lookup()函数回溯(backtrace)部分的处理，其他内容之后再补充，另外或许A7可以匹配到A6，这部分内容不太确定，完后再做确认。

<!-- more -->
#### 源码解读
所有源码来自[kernel.org](https://www.kernel.org/)，本文并未将该函数的全部内容列出
```c
/**********************************
 *	内核版本3.10
 *	函数位置：/net/ipv4/fib_trie.c	1405行
 *	函数说明：路由查表用函数，
 *	调用过程：ip_route_input_slow()->fib_lookup()->fib_table_lookup()
 *********************************/
int fib_table_lookup(struct fib_table *tb, const struct flowi4 *flp,
		     struct fib_result *res, int fib_flags)
{
@block_1：
	note:所需数据结构定义以及初始化
	struct tnode *pn;	//用作父节点	parent node;
	t_key key = ntohl(flp->daddr);	//待查找的地址
	t_key cindex = 0;	//子节点的索引号	child index
	struct tnode *cn;	//用作子节点	child node;
@block_2:
	note:首先从根节点开始，n为null，则直接到(failed)去处理，返回1,退出查找
	n = rcu_dereference(t->trie);
	if (!n)
		goto failed;
@block_3:
	note:根为leaf，则直接检查leaf的信息，之后跳转到(found)去处理，返回ret，退出查找
	if (IS_LEAF(n)) {
		ret = check_leaf(tb, t, (struct leaf *)n, key, flp, res, fib_flags);
		goto found;
	}
@block_4:
	note:该部分处理n为tnode
	pn = (struct tnode *) n;	//n为一个tnode，接下来要把这个tnode作为一个parent node来用，然后，查找它的child node
	chopped_off = 0;	//这个变量用来保存不匹配的位数，不匹配的内容，我们需要把他切掉，

	while (pn) {
		pos = pn->pos;	//该node所匹配的可匹配地址的起始位，
		bits = pn->bits;	//表示起始位之后的位数

		if (!chopped_off)
			cindex = tkey_extract_bits(mask_pfx(key, current_prefix_length),
							 pos, bits);	//取出bits，即child的索引
		n = tnode_get_child_rcu(pn, cindex);	//根据cindex取得child

@block_4_1:
		if (n == NULL) {
			goto backtrace;
		}
@block_4_2:
		if (IS_LEAF(n)) {	//child 是一个leaf，进行处理
			ret = check_leaf(tb, t, (struct leaf *)n, key, flp, res, fib_flags);
			if (ret > 0)
				goto backtrace;
			goto found;
		}
@block_4_3:
		cn = (struct tnode *)n;	//child 是一个node,进行处理

		if (current_prefix_length < pos+bits) {
			if (tkey_extract_bits(cn->key, current_prefix_length,
					cn->pos - current_prefix_length)
					|| !(cn->child[0]))
				goto backtrace;
		}

		pref_mismatch = mask_pfx(cn->key ^ key, cn->pos);

		if (pref_mismatch) {
			/* fls(x) = __fls(x) + 1 */
			int mp = KEYLENGTH - __fls(pref_mismatch) - 1;

			if (tkey_extract_bits(cn->key, mp, cn->pos - mp) != 0)
				goto backtrace;

			if (current_prefix_length >= cn->pos)
				current_prefix_length = mp;
		}

		pn = (struct tnode *)n; /* Descend */
		chopped_off = 0;
		continue;

backtrace:
		chopped_off++;

		/* As zero don't change the child key (cindex) */
		while ((chopped_off <= pn->bits) && !(cindex & (1<<(chopped_off-1))))
			chopped_off++;

		/* Decrease current_... with bits chopped off */
		if (current_prefix_length > pn->pos + pn->bits - chopped_off)
			current_prefix_length = pn->pos + pn->bits - chopped_off;


		/*
		 * Either we do the actual chop off according or if we have
		 * chopped off all bits in this tnode walk up to our parent.
		 */
		if (chopped_off <= pn->bits) {
			cindex &= ~(1 << (chopped_off-1));
		} else {
			struct tnode *parent = node_parent_rcu((struct rt_trie_node *) pn);
			if (!parent)
				goto failed;

			/* Get Child's index */
			cindex = tkey_extract_bits(pn->key, parent->pos, parent->bits);
			pn = parent;
			chopped_off = 0;

			goto backtrace;
		}
	}

failed:
	ret = 1;
found:
	rcu_read_unlock();
	return ret;
}
```


#### 已有路由项
标号 | 地址 | 1 | 2 | 3 | 4
---- | ---- | ---- | ---- | ---- | -----
A1 | 192.168.10.0/24 | 11000000 | 10101000 | 00001010 | 00000000
A2 | 192.168.20.0/24 | 11000000 | 10101000 | 00010100 | 00000000
A3 | 2.232.20.0/24 | 00000010 | 11101000 | 00010100 | 00000000
A4 | 192.168.20.32/30 | 11000000 | 10101000 | 00010100 | 00100000
A5 | 192.168.20.4/30 | 11000000 | 10101000 | 00010100 | 00000100
A6 | 0.0.0.0/0 | 00000000 | 00000000 | 00000000 | 00000000

#### 路由树结构
* (pos=0,bits=3)
  * 000   (pos=6,bits=1)
    * 0   [A6]
    * 1   [A3]
  * 110   (pos=19,bits=2)
    * 01   [A1]
    * 10   (pos=26,bits=1)
      * 1   [A4]
      * 0   (pos=29,bits=1)
        * 0   [A2]
        * 1   [A5]

#### 待查找ip
标号 | 地址 | 1 | 2 | 3 | 4
---- | ---- | ---- | ---- | ---- | -----
A7 | 64.X.X.X/24 | 01000000 | X | X | X

#### 查找过程
1. 首先从block_2开始，且根不为null，之后到block_3；
2. 根是一个node，到block_4处理；
3. 先看一下用到的几个变量（chopped_off = 0; pos =0; bits = 3；key = 64 X X X ）
4. 根据pos、bits、key可以得到cindex = b010；
5. 根据路由树结构，我们可以看出，该根node只有两个child，分别是[000]和[110]，因此对应cindex为空，则到block_4_1来处理。
6. 发现没有child，会直接跳转到backtrace（回溯）来进行处理，这里解释一下回溯处理的目的，通过回溯来查找与cindex所指向的空child同一父node下的其他child的中能最长前缀匹配cindex的child。
7. 先来直观的看一下cindex是如何与其他child最长前缀匹配的，cindex = [010];其他child = [000]、[110]，从左起第一位开始，[000] 和 [010] 都为0，左起第二位，不匹配了，所以这里最长匹配到第1位。假设有个child = [011]，则最长匹配到第2位。
8. 下面给出backtrace处理过程中关键值的变化，pn->bits = 3没变

说明 | chopped_off | cindex | chopped_off <= pn->bits | !(cindex & (1<<(chopped_off-1)))
---- | ---- | ---- | ---- | -----
while()条件处理 | 1 |  010b | 真 | !(010b & 001b) 结果为非0
while内处理 | 2
while()条件处理 | 2 | 010b | 真 | !(010b & 010b) 结果为0

注释：while循环干了什么？
找到这bits位，最后一位为1的位置。那么找到了，就是右起第2位，之后会把这位值修改为0，设置成新cindex，通过{ cindex &= ~(1 << (chopped_off-1)) };完成修改。当前的前缀长度为1（current_prefix_length = 1 ）

9. 好了现在我们有新的cindex了，再返回while(pn)循环；
10. n = tnode_get_child_rcu(pn, cindex);该函数找到child，这次child是个leaf;
11. 进入block_4_2去处理，check_leaf()返回的ret会大于0（注释：待补充解释），然后再次到backtrace去处理；
12. 同样看相关值变化

说明 | chopped_off | cindex | chopped_off <= pn->bits | !(cindex & (1<<(chopped_off-1)))
---- | ---- | ---- | ---- | -----
while()条件处理 | 3 |  000b | 真 | !(000b & 001b) 结果为非0
while内处理 | 4
while()条件处理 | 4 | 000b | 假 | 不考虑
13. current_prefix_length = pn->pos + pn->bits - chopped_off；结果为-1；
14. if (chopped_off <= pn->bits)不满足此条件，会到cindex父node的父node，再继续匹配最长前缀，node_parent_rcu()获取到的父node为空，因此跳转到failed执行，退出。
