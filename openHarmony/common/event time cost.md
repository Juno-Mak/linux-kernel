# event time cost


### proc cost
新建一个空的proc节点，循环对齐进行 open read write close的操作
分别记录下每个操作的 最大值 最小值 平均值
		max		min		avg
open	566146	1562	2004
read	452083	0		279
write	408854	0		233
close	403125	0		403