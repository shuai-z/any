# v8 object数据结构与算法

其实并不一定得看源码才能知道对象的数据结构，快照里也能看。

## 1. heap snapshot

做几个实验，先从最简单的object literal开始，

### 1.1. object literal

```js
let o0 = {};
let o1 = {p1:1};
let o2 = {p1:1, p2:2};
let o3 = {p1:1, p2:2, p3:3};
```

结果（从上到下为o0,o1,o2,o3）：

<img src=assets/v8/object-memory-1.png height=160 />

注意观察Shallow Size，如果忽略空对象o0的话，可以大胆猜测:

> `object_size = 12 + property_number * 4`
>
> (4个字节，因为是32位指针)


验证一下，

```js
let o4 = {p1:1, p2:2, p3:3, p4:4}; // = 28 = 12+4*4
let o5 = {p1:1, p2:2, p3:3, p4:4, p5:5}; // = 32 = 12+5*4
let o10 = {p1:1, p2:2, p3:3, ..., p10:10}; // = 52 = 12+10*4

let o128 = {p1:1, p2:2, p3:3, ..., p128:128}; // = 524 = 12+128*4

// 当超过128个属性时，结构发生变化
let o129 = {p1:1, p2:2, p3:3, ..., p129:129}; // = 12 ???
```

从snapshot可以看到，到129个属性的时候，object变成了12个字节，此时多显示了一个`properties`，`properties`自身有524个字节，能猜到应该是个数组。

```
524 = 4 // map指针
    + 4 // 数组长度length
    + 129*4
```

<img src=assets/v8/object-memory-2.png height=120 />

<img src=assets/v8/object-memory-3.png height=250 />

[代码](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/heap/factory.cc;drc=8581d300388ba3d752e73289dcb702625119e59f;bpv=1;bpt=1;l=3241)

顺便一起说了，当超过1020个属性的时候，会发生一次大的改变（可以从snapshot观察）：

- descriptor不再放到map里（而是一起放进properties）；
- properties的结构从 *数组（FixedArray）* 变成 *字典（NameDictionary）*；

目前已经看到三种方式：

1. 直接在对象里 - in-object
2. 在数组里 - out-of-object
3. 在字典里 - slow

-- DONE --

再看空对象的问题：为什么空对象需要28个字节，它减去12之后还剩下16个字节，有什么特殊的作用吗？

我开始也好奇这多的16个字节能让空对象拥有什么神奇的功能吗，没有，所以只能猜到一种可能：这16个字节是 *预先* 分配的(inobject)空位，可以容纳未来的4个属性值。那就实验一下看看吧。

### 1.2. 动态添加新属性

```js
let o = {a:1};
o.b = 2;
```

新添加的属性值`b=2`会追加到对象末尾（inobject）吗？

不会。因为一开始就只分配了12+4=16个字节给这个object，如果还要加到对象末尾的话，就只能重新分配一块新的内存了。而重新分配也意味着指向这块内存的指针也得更新。

那为什么不一次多申请点空间呢？不过这样的话可能会浪费更多空间。

```js
// 比较两种代码

let o = {p:1};
o.p1=1;
o.p2=2;
o.p3=3;
...

let o = {a:1,b:2,c:3};
```

但是空对象不一样，`{}`之后肯定极大概率会往里面添加属性，所以事先留了几个空。


## 2. 数据结构

相关文件都在 src/objects 下。

[src/objects/js-objects.h](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/objects/js-objects.h)

[src/objects/js-objects.tq](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/objects/js-objects.tq)

```tq
// JSReceiver corresponds to objects in the JS sense.
@abstract
@highestInstanceTypeWithinParentClassRange
extern class JSReceiver extends HeapObject {
  properties_or_hash: FixedArrayBase|PropertyArray|Smi;
}

type Constructor extends JSReceiver;

@generateCppClass
@apiExposedInstanceTypeValue(0x421)
@highestInstanceTypeWithinParentClassRange
extern class JSObject extends JSReceiver {
  // [elements]: The elements (properties with names that are integers).
  //
  // Elements can be in two general modes: fast and slow. Each mode
  // corresponds to a set of object representations of elements that
  // have something in common.
  //
  // In the fast mode elements is a FixedArray and so each element can be
  // quickly accessed. The elements array can have one of several maps in this
  // mode: fixed_array_map, fixed_double_array_map,
  // sloppy_arguments_elements_map or fixed_cow_array_map (for copy-on-write
  // arrays). In the latter case the elements array may be shared by a few
  // objects and so before writing to any element the array must be copied. Use
  // EnsureWritableFastElements in this case.
  //
  // In the slow mode the elements is either a NumberDictionary or a
  // FixedArray parameter map for a (sloppy) arguments object.
  elements: FixedArrayBase;
}
```

[src/objects/heap-object.tq](https://source.chromium.org/chromium/chromium/src/+/master:v8/src/objects/heap-object.tq)

```tq
@abstract
extern class HeapObject extends StrongTagged {
  const map: Map;
}
```

也没什么对吧，其实要看代码的话map和descriptor的数据结构更重要。

### 2.1. 几个数字

- 空对象和 **128**：

https://source.chromium.org/chromium/chromium/src/+/master:v8/src/heap/factory.cc;drc=7dc15fb114a4483abae541847b9abc311cbe8eda;l=3241

```cc
Handle<Map> Factory::ObjectLiteralMapFromCache(Handle<NativeContext> context,
                                               int number_of_properties) {
  if (number_of_properties == 0) {
    // Reuse the initial map of the Object function if the literal has no
    // predeclared properties.
    return handle(context->object_function().initial_map(), isolate());
  }

  // Use initial slow object proto map for too many properties.
  const int kMapCacheSize = 128;
  if (number_of_properties > kMapCacheSize) {
    return handle(context->slow_object_with_object_prototype_map(), isolate());
  }
  
  // ...
```

- 空对象的 **4** 个空位：

https://source.chromium.org/chromium/chromium/src/+/master:v8/src/objects/js-objects.h;drc=7dc15fb114a4483abae541847b9abc311cbe8eda;bpv=1;bpt=1;l=752

```cc
// This constant applies only to the initial map of "global.Object" and
// not to arbitrary other JSObject maps.
static const int kInitialGlobalObjectUnusedPropertiesCount = 4;
```

- 1020

https://source.chromium.org/chromium/chromium/src/+/master:v8/src/objects/property-details.h;l=190;bpv=0;bpt=1

```cc
static const int kDescriptorIndexBitCount = 10;
static const int kFirstInobjectPropertyOffsetBitCount = 7;
// The maximum number of descriptors we want in a descriptor array.  It should
// fit in a page and also the following should hold:
// kMaxNumberOfDescriptors + kFieldsAdded <= PropertyArray::kMaxLength.
static const int kMaxNumberOfDescriptors = (1 << kDescriptorIndexBitCount) - 4;
static const int kInvalidEnumCacheSentinel =
    (1 << kDescriptorIndexBitCount) - 1;
```


## 3. 算法

知道了数据结构，所以`obj.key`怎么从`key`找到索引位置呢？

（当然，肯定是有缓存的{hash->index}）

map里有个instance_descriptors，（在超过限制之前）属性会加到这里面：

```tq
@generatePrint
@generateCppClass
extern class EnumCache extends Struct {
  keys: FixedArray;
  indices: FixedArray;
}

@export
struct DescriptorEntry {
  key: Name|Undefined;
  details: Smi|Undefined;
  value: JSAny|Weak<Map>|AccessorInfo|AccessorPair|ClassPositions;
}

@generateCppClass
extern class DescriptorArray extends HeapObject {
  const number_of_all_descriptors: uint16;
  number_of_descriptors: uint16;
  raw_number_of_marked_descriptors: uint16;
  filler16_bits: uint16;
  enum_cache: EnumCache;
  descriptors[number_of_all_descriptors]: DescriptorEntry;
}
```

这个结构在heap snapshot里能看到一部分。

还是看个例子吧，



