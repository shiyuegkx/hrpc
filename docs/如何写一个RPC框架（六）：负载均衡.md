> 在后续一段时间里， 我会写一系列文章来讲述如何实现一个RPC框架。 这是系列第六篇文章， 主要讲述了RPC中负载均衡这一块的内容。

## 常见的LB策略
常见的LB策略有很多：

 1. RoundRobin (RR)： 一个列表中轮着来
 2. WeightedRoundRobin (WRR)： 带权重的RR
 3. LocalFirst：本地服务优先
 4. Random：随机选择
 5. ConsistentHash： 一致性哈希

这些策略中，除了最后一个，别的都很好理解。 这里就给一些一致性哈希负载均衡相关的原理和实现吧。

## 一致性哈希负载均衡
### 一致性哈希算法
首先，我们需要去了解什么是一致性哈希算法。网上有很多关于一致性哈希的文章， 在这里我推荐一篇写的比较细致且容易理解的：[一致性哈希算法](http://www.zsythink.net/archives/1182)

### 为什么要用一致性哈希来做负载均衡
假设我们根据userid来做hash，在服务器数量发生变动时，只有少数用户的请求会发送到新的机器上， 这样可以最大程度的利用服务器的本地缓存。

### 简单实现
一致性哈希算法在我看来只有两个注意点：

 1. 如何找到某一个hash值在环中的下一个节点
 2. 如何实现虚拟节点的replica

这两个问题比较好解决：

 1. 对于第一个问题， 可以利用treemap来实现
 2. 对于第二个问题， 我们只需要对某一个物理节点的key做多次的【修改key，再hash，再存入treemap中】即可

对应的代码如下：
```
public interface HashFunction<T> {
	int hash(T t);
}

public class ConsistentHash<T> {

	private final HashFunction hashFunction;
	private final int numberOfReplicas;
	private final SortedMap<Integer, T> circle = new TreeMap<>();

	public ConsistentHash(HashFunction hashFunction, int numberOfReplicas,
			Collection<T> nodes) {
		this.hashFunction = hashFunction;
		this.numberOfReplicas = numberOfReplicas;

		for (T node : nodes) {
			add(node);
		}
	}

	public void add(T node) {
		for (int i = 0; i <numberOfReplicas; i++) {
			circle.put(hashFunction.hash(node.toString() + i), node);
		}
	}

	public void remove(T node) {
		for (int i = 0; i <numberOfReplicas; i++) {
			circle.remove(hashFunction.hash(node.toString() + i));
		}
	}

	public T get(Object key) {
		if (circle.isEmpty()) {
			return null;
		}
		int hash = hashFunction.hash(key);
		if (!circle.containsKey(hash)) {
			SortedMap<Integer, T> tailMap = circle.tailMap(hash);
			hash = tailMap.isEmpty() ? circle.firstKey() : tailMap.firstKey();
		}
		return circle.get(hash);
	}
}
```

