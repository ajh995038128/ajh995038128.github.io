---
layout: post
title: 在java中弱引用，软引用，虚引用和强引用的不同之处
date: 2015-05-28 11:56:22
categories: java 翻译
excerpt: java 引用
---

原文地址：http://javarevisited.blogspot.jp/2014/03/difference-between-weakreference-vs-softreference-phantom-strong-reference-java.html （可能需要梯子才能看到）

首先，由于这是我的第一篇翻译文章，再加上我的英语水平和技术水平有限在翻译上难免会有不妥之处，敬请见谅。


弱引用（WeakReference）和软引用（softReference）被加入到java api中已经很长一段时间了，但并不是每一位java 程序员都对它们很熟悉。也就意味着在哪和怎么使用弱引用和软引用之间有一道间隙。引用类（Reference Classes）在文章《GC 如何工作》中特别重要。我们都知道GC回从那些应该被回收的对象上回收内存，但是不是很多的程序员知道这个“应该”是基于什么样的引用指向这个对象来决定的。这也是Java中的软引用和弱引用的主要不同。GC很可能回收一个对象，如果它只有弱引用指向它，另一方面，当java虚拟机特别需要内存的时候，那些软引用的对象只能眼巴巴的被回收。软引用和弱引用的这些特别的行为让它们在一些特定的场景特别有用。例如：软引用用于做缓存看起来特别棒，所以当java虚拟机需要内存的时候它把那些只有软引用指向的对象随意出。而相对应的弱应用用于保存元数据(Meta data)特别棒。例如:保存ClassLoader的引用，如果没有类被加载那么就没有指针保持ClassLoader的引用，一个弱引用让ClassLoader能够在强引用被移除时立刻被GC回收。在这篇文章中我们会探讨更多java中各种引用如：强引用和虚引用。

WeakReference vs. SoftReference

为那些不知道java中有四种引用的同学：

1.强引用（Strong Reference）

2.弱引用（Weak Reference）

3.软引用 （Soft Reference）

4.虚引用 （Phantom Reference）


强引用是我们每天编程生活中使用最简单的，例如：在代码 String s = “abc”,引用对象s有一个强引用一个String对象 “abc”。任何有强引用的对象都不应该被GC回收。显然这些是java程序需要的对象。弱引用用java.lang.ref.WeakReference类表示，你可以通过使用以下代码创建一个弱引用：

{% highlight java %}
Counter counter = new Counter(); //强引用 line 1  

WeakReference<Counter> weakCounter = new WeakReference<Counter>（counter）;//弱引用  

counter = null;//现在Counter对象能够被GC回收了  
｛% endhighlight %｝
 现在，一旦你让强引用counter = null，line 1被创建的对象变成了可被GC回收了。因为它没有任何的强引用并且被引用对象weakCounter弱引用不能阻止Counter对象被GC回收。另一方面，如果它被软引用着，Counter对象将不会被回收直到ava虚拟机特别需要内存。软引用在java中用java.lang.ref.SoftReference类表示。你可以通过以下代码创建一个软引用：

 {% highlight java %}
Counter prime = new Counter();//prime 持有一个强引用 line 2  

SoftReference<Counter> soft = new SoftReference<Counter>(prime);// 软引用变量有一个指向line 2倍创建的Counter 对象  

prime = null ; //现在Counter对象只有在jvm特别需要内存的时候才会被回收  
｛% endhighlight %｝
让强引用变成null之后，line2创建的Counter对象只有一个不能阻止它被垃圾回收但是能延迟回收的软引用，而在弱引用的情况下会更快。由于软引用和弱引用这个主要不同，软引用更适合于做缓存，而弱引用韩庚适合保存元数据。一个弱引用的例子是WeakHashMap，它是像HashmMap或者TreeMap的其他的Map接口的实现，但是它有一个独一无二的特性:WeakHashMap把它的key包装成弱引用也就是意味着一旦真是对象的强引用被移除，在WeakHashMap内部的弱引用不能保证他们不被垃圾回收。
虚引用是在java.lang.ref包中可以得到的第三种引用。虚引用用java.lang.ref.PhantomReference类表示。那些虚引用指向的对象可以随时被回收如果gc喜欢。与弱引用和软引用相似的，你可以通过以下代码创建一个虚引用：


{% highlight java %}
DigitalCounter digit = new DigitalCounter();//digit 引用变量有一个强引用 line3

PhantomReference<DigitalCounter> phantom = new PhantomReference<DigitalCounter>(digit);//line3 创建的对象的虚引用  

digit = null;  
｛% endhighlight %｝
一旦你移除了强引用，由于只有一个不能阻止被回收的虚引用指向他，line3创建的对象能随时被垃圾回收。
除了，熟知的WeakReference，SoftReference，PhantomReference和WeakHashMap，还有一个叫ReferenceQueue的类很值得了解。当创建任何弱引用，软引用和虚引用时，你可以提供一个ReferenceQueue的实例，如下面代码展示的一样：


{% highlight java %}
ReferenceQueue reQueue= new ReferenceQueue();// 引用会被保存在这个队列中来清理  

DigitalCounter = digit =new DigitalCounter();  

PhantomReference<DigitalCounter> phantom = new PhantomReference<DigitalCounter>(digit,reQueue);  
｛% endhighlight %｝
引用的实例会被追加到ReferenceQueue里，你可以使用它进行任何清理通过poll 引用队列，一个对象的生命周期通过下面的图能得到很好的总结：
  [对象生命周期]({{ site.url }}/images/20150526201834341.png)
这就是java中弱引用和软引用所以的不同了。我们也学习到了一些基本的引用类，如java中的弱引用，软引用和虚引用，还有WeakHashMap，ReferenceQueue.小心使用引用能帮助垃圾回收期更好的工作和得到更好的java内存管理。

谢谢，大家以上就是这篇文章的内容。如果有什么翻译不好的地方欢迎在评论中告诉我。
