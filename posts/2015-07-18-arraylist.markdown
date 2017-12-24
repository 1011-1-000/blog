---
layout: post
title: "看ArrayList时的问题总结"
date: 2015-07-18 21:32:33 +0800
comments: true
categories: java
tags: [java,arrayList]
---

问题：将一个List中所有的偶数值删除。

- 解法一：
```java
	public static void removeEvenVerl(List<Integer> lst){
		int i = 0;
		while(i<lst.size()){
			if(lst.get(i)%2==0){
				lst.remove(i);
			}
			else{
				++i;
			}
		}
	}
```
这是一个效率很不好算法，因为对于ArrayList来说，lst.remove的代价是很高的。而对于LinkedList来说lst.get的代价也是很高。
<!--more-->

- 解法 二：应该不能算是一个解法，因为程序会异常。
```java
	public static void removeEvenVerl(List<Integer> lst){
		for(Integer x : lst){
			if(x % 2 == 0){
				lst.remove(x);
			}
		}
	}
```
这种方法看起来好像没有什么问题，但是这种增强的for循环，编译器最后会转为Iterator来处理，而lst.remove会使得正在被迭代的集合结构发生改变，而这对于Iterator来说是非法的，因此程序异常。

- 解法三：
```java
	public static void removeEvenVerl(List<Integer> lst){
		Iterator<Integer> itr = lst.iterator();
		while(itr.hasNext()){
			if(itr.next()%2==0){
				itr.remove();
			}
		}
	}
```
这种解法才是OK的。


另一个问题：为什么这些集合类一定是实现Iterable接口而不是Iterator接口？
因为Iterator接口的核心方法是next和hasNext，而这两个方法是依赖于迭代器的当前位置的，如果直接实现Iterator接口，会导致集合对象中包含当前迭代位置的数据。当集合在不同的方法间被传递时，会出现每个调用都是从一个共同的迭代位置开始。而用Iterable则会有多个迭代器，不会出现迭代器相互干扰的问题。

