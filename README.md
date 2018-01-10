# LinkedHashMap_LruCache_Note
从源码分析LinkedHashMap以及其在LruCache中的应用

在之前的文章[红黑树在HashMap中的应用](http://blog.csdn.net/troy_kfrozen/article/details/78906022)中，我们分析了HashMap的实现原理以及查找，插入和删除操作的源码，这一篇我们就来看看HashMap的一个子类：LinkedHashMap。

- **LinkedHashMap**

	**LinkedHashMap**继承自HashMap并实现了Map接口，它的API与HashMap完全一致，用法也大致相同，同样是非线程安全的集合。我们	知道HashMap存储的节点都是HashMap.Node类型的，到了LinkedHashMap中，节点类型变成了LinkedHashMapEntry，这是Node的一	个子类，在Node类的基础上增加了before和after两个指针，用于双向链表的实现，下面代码是LinkedHashMapEntry类的定义：

	```
 	/**
 	* HashMap.Node subclass for normal LinkedHashMap entries.
 	*/
 	static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
    	LinkedHashMapEntry<K,V> before, after;
    	LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
        	super(hash, key, value, next);
    	}
 	}
	```

	由于这两个指针的存在，LinkedHashMap可以在保持HashMap存储结构不变的前提下，将持有的节点额外以双向链表的形式连接起来。这个	双向链表也是LinkedHashMap与其父类最大的不同，虽然存储方式完全相同，但是LinkedHashMap可以通过双向链表做到有序遍历，而这	个顺序取决于**accessOrder**这个成员变量。

	首先，老规矩，一起先来看看LinkedHashMap的全局成员变量：

	```
 		/**
     	* The head (eldest) of the doubly linked list.
     	*/
    	transient LinkedHashMapEntry<K,V> head; //指向双向链表头结点的指针

    	/**
    	 * The tail (youngest) of the doubly linked list.
     	*/
    	transient LinkedHashMapEntry<K,V> tail; //指向双向链表尾结点的指针

    	/**
     	* The iteration ordering method for this linked hash map: <tt>true</tt>
     	* for access-order, <tt>false</tt> for insertion-order.
     	*
     	* @serial
     	*/
    	//LinkedHashMap对于双向链表节点的顺序有两层维护方式。
    	//第一层：按照节点插入顺序排序，先来先排
    	//第二层：若accessOrder变量被置位true（在构造函数中传入），则每一次对节点的访问或修改都会将该节点移动到队尾
    	//这样一来head指针指向的就是最久没有被访问的节点，这一个特性完全符合LRU(Least Recently Used)算法
    	final boolean accessOrder; 
	```

	OK，成员变量不多，作用也很直观。另外值得注意的是LinkedHashMap所重写的HashMap中留下的几个钩子方法，这些方法都是在	HashMap结构发生变化或节点被访问时被调用（例如put，get，remove），而HashMap中这些都是空方法，LinkedHashMap通过重写这	些方法实现了对节点双向链表结构的维护。下面就一起来看看这几个钩子方法的实现：

	```
	//这个方法是在节点被访问或者被修改时被调用的
	void afterNodeAccess(Node<K,V> e) { // move node to last
        	LinkedHashMapEntry<K,V> last;
        	if (accessOrder && (last = tail) != e) { //若accessOrder为false或e已经在链表尾部，则无需调整
            	//暂存被操作节点e为p，其前驱节点为b，后继节点为a
            	LinkedHashMapEntry<K,V> p =
                	(LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
            	p.after = null; //将p的后继置空
            	if (b == null)
                	head = a; //若e的前驱节点为空，则说明e之前为链表的头结点，现在将e原先的后继结点变为头结点
            	else
                	b.after = a; //前驱不为空，则连接e的前驱和后继节点，将e从链表中断开
            	if (a != null)
                	a.before = b; //e原先的后继不为空，则与原先的前驱b连接
            	else
                	last = b; //否则的话将尾指针指向b
            	if (last == null)
                	head = p; //若尾指针此时为空，说明e是链表中唯一的节点，则将头指针重新指向它
            	else {
                	p.before = last; //否则的话将其移到链表尾部
                	last.after = p;
            	}
            	tail = p; //尾指针指向尾节点
            	++modCount; //结构操作数加一
        	}
	}
    
	//这个方法是在有新节点插入时被调用的
	//evict这个变量如果为true，说明需要在节点添加后进行一次eldest节点删除。反之则是单增长模式
	void afterNodeInsertion(boolean evict) { // possibly remove eldest
    	LinkedHashMapEntry<K,V> first;
    	//如果不为单增长模式，链表不为空，且removeEldestEntry的返回值为true，就将头结点删除
    	//值得注意的是removeEldestEntry方法默认的返回是false，即LinkedHashMap默认插入时不删除eldest节点
    	//若需修改，则在子类中重写该方法
    	if (evict && (first = head) != null && removeEldestEntry(first)) {
        	K key = first.key;
        	removeNode(hash(key), key, null, false, true); //删除链表头结点
    	}
	}

	//这个方法当有节点被删除时会被调用
	//e为待删除节点，这里做的也很简单，就是将这个已经被HashMap存储结构删除的节点从双向链表中移除
	void afterNodeRemoval(Node<K,V> e) { // unlink
    	LinkedHashMapEntry<K,V> p = (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
    	p.before = p.after = null;
    	if (b == null)
        	head = a;
    	else
        	b.after = a;
    	if (a == null)
        	tail = b;
    	else
        	a.before = b;
	}
	```

	上面这三个就是LinkedHashMap重写的其父类的三个钩子方法，除了这三个方法外，LinkedHashMap还重写了其他的一些HashMap的方	法，其主要目的有两个，一是将节点类从Node包装为LinkedHashMapEntry， 二是维护自身的双向链表。下面我们就一起来看下这几个被	重写的方法：

	```
	//这里用LinkedHashMapEntry代替了HashMap.Node作为链表节点
	Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    	LinkedHashMapEntry<K,V> p = new LinkedHashMapEntry<K,V>(hash, key, value, e);
    	linkNodeLast(p); //将新创建出的节点添加到链表尾部
    	return p;
	}

	//这是构建桶中的红黑树结构时用到的新建节点的方法，这里相比HashMap会多做一步linkNodeLast，即将节点添加到链表尾部
	TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
    	TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
    	linkNodeLast(p);
    	return p;
	}

	//这个方法在untreeify过程中会被调用，作用就是将传入的TreeNode类型的p转换为普通的Node类型，当然这里是LinkedHashMapEntry
	//需要说明的是，红黑树的节点类型TreeNode其实是LinkedHashMapEntry的子类，所以这里直接进行了强转
	Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
    	LinkedHashMapEntry<K,V> q = (LinkedHashMapEntry<K,V>)p;
    	LinkedHashMapEntry<K,V> t = new LinkedHashMapEntry<K,V>(q.hash, q.key, q.value, next);
    	transferLinks(q, t); //这里是一个替换操作，用t替换原先链表中的q
    	return t;
	}

	//这个方法会在treeifyBin方法中被调用，作用是将节点类型由LinkedHashMapEntry转换成TreeNode
	TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    	LinkedHashMapEntry<K,V> q = (LinkedHashMapEntry<K,V>)p;
    	TreeNode<K,V> t = new TreeNode<K,V>(q.hash, q.key, q.value, next);
    	transferLinks(q, t); //与上相同，替换双向链表中的节点
    	return t;
	}

	//LinkedHashMap还重写了父类中的get方法
	public V get(Object key) {
    	Node<K,V> e;
    	if ((e = getNode(hash(key), key)) == null) //前半部分跟HashMap完全一样，也是通过getNode方法完成
        	return null;
    	if (accessOrder) //若key匹配到了对应得value，且accessOrder为true
        	afterNodeAccess(e); //通过afterNodeAccess方法将被访问节点移动到链表尾部
    	return e.value;
	}
	```

	可以看到，LinkedHashMap并没有改变任何HashMap中的存储结构和操作，只是将Node节点替换为了LinkedHashMapEntry节点，并且	在每一个操作方法后都会对自身的双向链表进行相应的维护。从这个角度看来，LinkedHashMap同时具有了Map和链表的特性，而这样的特	性使得它在很多地方都被应用，比如下面要出场的LruCache。
	
- **LruCache**

	首先，LRU是Least Recently Used的缩写，也即最少使用算法。而LruCache，顾名思义就是基于LRU算法思想的缓存策略。它会将有限的缓存对象以强引用的方式保存，当其中某一个缓存对象被访问后，该对象就会被升级一次，而当队列已满且有新的添加请求时，最久没有被访问过的那个对象就会被移除出缓存队列，LruCache对其的强引用也会断开，以方便可能的GC。
	
	看到这里，是不是会有一种LinkedHashMap简直就是为了LRU算法而生的感觉😆。。 言归正传，Android对于LruCache的实现确实就是基于LinkedHashMap完成的，上面所说的对象被访问后升级其实就是LinkedHashMap中afterNodeAccess方法所做的事情——将该节点移动至链表尾部，这样一来，链表的头结点必定就是最久没有被访问的对象，也即LruCache需要腾地方时要移除的对象。下面先一起来看下LruCache的构造函数：
	
	```
	//maxSize是你所需要的缓存队列的容量
	public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        //重点在这里，map成员变量是一个LinkedHashMap实例，其初始容量为0，扩容因子为0.75
        //accessOrder为true，即节点被访问后需要调整位置到链表尾部
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
    
    //顺带看一下这个扩容方法，与其他的集合类不同，LruCache并不存在自动扩容机制，这个resize方法也是由外界调用的
    //如果你需要更大或更小的缓存队列，可以调用这个public的resize方法
    public void resize(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }

        synchronized (this) { //对象锁，线程安全
            this.maxSize = maxSize;
        }
        trimToSize(maxSize); //这个方法很重要，用于调整LruCache缓存队列的容量，下面会有详述
   }
	```
	
	构造函数很简单，记录一下最大缓存容量，创建一个初始容量为0的LinkedHashMap。下面我们要重点来看看trimToSize这个内部方法，LruCache缓存队列容量的变化都是靠它完成的，put，get和resize等方法中都会对其进行适时的调用。
	
	```
	 //这里的输入参数maxSize指的是目标容量，方法体中的size变量是当前持有的缓存对象数
	 public void trimToSize(int maxSize) {
        while (true) { //开启循环
            K key;
            V value;
           
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) { //size的合法性检查
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }
                //若当前持有的缓存数量还未到设置的最大容量，则退出循环，结束
                if (size <= maxSize) {
                    break;
                }
                
                //这里用到了LinkedHashMap中的eldest方法，该方法会返回LinkedHashMap维护的双向链表的头结点
                //也即最久没有被访问过的节点。注意这个方法是被@hide注解的，且从注释来看，是专门为Android添加的
                Map.Entry<K, V> toEvict = map.eldest();
                if (toEvict == null) {
                    break; //若取到的对象为空，即map为空，则没必要继续进行调整
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key); //从map中删除之前取出的最老节点
                size -= safeSizeOf(key, value); //当前存储的节点数减一
                evictionCount++; //移除的缓存个数加一
            }
			  //钩子方法，在这里可以对被移除的节点做一些事情，在LruCache中为空方法，由子类按需实现
            entryRemoved(true, key, value, null); 
        }
    }
	```
	翻看LruCache的源码我们会发现它的put，get等操作其实都是通过LinkedHashMap类型的map变量来完成的，因此它对于缓存对象的存储结构与LinkedHashMap完全一致，并且得益于LinkedHashMap内部维护的双向链表轻松实现了LRU算法的思想。
	
	最后再提一句，LruCache中所有的public方法都含有对象锁，因此它是线程安全的。但是在高并发的情况下其性能不会很好，因为在一条线程获得锁时，不论它操作的是哪一个方法，该LruCache对象中的所有其他public方法都会被锁住而对其他线程不可用。
	
	感谢阅读！
	
**版权声明：原创不易，转载前请留言获得作者许可，转载后标明作者 Troy.Tang 与 原文链接。**
	
