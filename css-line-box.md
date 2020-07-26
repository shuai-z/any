## line-height, font-size, vertical-align

* 前提：inline formatting context, line box, inline box

主要写line box的高度和内部inline box的对齐。

关键点：

1. line box的高度要能足够容纳其内部的盒子，也就是说它的高度至少要达到内部盒子的最大高度。
2. 对于img,inline-block这样的元素，高度是他们的margin box；对于普通的inline box，他们的高度是line-height。
3. 确定baseline的位置。

### 1. 文字

每个字体都有font metrics计算文字的高度： A+D，A和D分别是baseline之上和之下的距离（不一定是ascender和descender）。

```html
<style>
span {
  outline: 1px solid;
}
</style>
<div style="font-size:36px;">
  <span>a</span>
  <span>b</span>
  <span>c</span>
  <span>中</span>
  <span>文</span>
</div>
```

<img src=assets/css/line-box-1.png height=100 />

图中虚线是baseline的位置，每个盒子的高度都是50（苹方字体，36px），这个高度是只与当前使用字体有关，与line-height无关。

汉字与英文字母不一样，CJK文字没有baseline，而是类似豆腐块，文字放在这个豆腐块的中间（或许也不一定完全居中），每个都一样宽高（比如36px*50px），并且与底对齐。

可以看出苹方的汉字在正中间，所以后面会看到当缩小line-height到font-size的时候（或者line-height:1），可以正好卡着汉字，
并且当line-height为0的时候，line box的顶部（或底部）正好在汉字的中间。


### 2. line-height，确定文字的高度

文字的高度由line-height决定。line-height可以比font metric的高度小，也就是文字占据的空间可能比需要的空间要小，当line-height:0的时候高度可以是0。

因为没有指定，上面line-height是normal，根据字体和大小，最终实际值是50px，此时，line height = AD。

line height = L + AD = L_before + AD + L_after

通过公式计算出leading（L），分成两半，一半在文字上方，一半在下方。leading可以是负数。

```html
<!-- 方便观察 -->
<div style="height:100px;background:rgba(0,0,0,0.2);"></div>
<div style="font-size:36px;outline: 1px solid;line-height:50px;"> <!-- 改变line-height，观察line box的高度 -->
  <!-- span继承font-size和line-height，所以以下都一样高，leading也一样 -->
  <span>a</span>
  <span>b</span>
  <span>c</span>
  <span>中</span>
  <span>文</span>
</div>
```

<img src=assets/css/line-box-2.png height=400 />


### 3. 确定baseline

一个line box的font-size和line-height可以确定初始的高度和baseline，此处想象有一个看不见的文字。

先考虑最简单的，所有元素line-height:normal, vertical-align:baseline。所以line box里面每个inline box的baseline都要跟这个line box的baseline对齐。

```html
<style>
  div {
    outline:1px solid #ccc;
  }
  span {
    outline: 1px solid #888;
  }
</style>
<div style="font-size:30px;">
  <span>看不到xxx</span>
  <span style="font-size:24px;">abc</span>
  <span style="font-size:16px;">abc</span>
</div>
```

<img src=assets/css/line-box-3.png height=50 />

图中能看到每个box的baseline与这个line box的baseline是对齐的，因为这两个inline box的高度比"看不到的文字"的高度要小，看起来比较“正常”。下面把字体改大一点。

如果还是直接按baseline对齐：

<img src=assets/css/line-box-4.png height=50 />

此时中间盒子的高度已经超出line box了，因为“line box的高度必须要能容纳内部所有盒子”，所以必须把line box“撑开”，结果就是baseline的位置要往下移。

<img src=assets/css/line-box-5.png height=50 />

现在可以把line-height考虑进来，先修改一下，方便查看盒子的真实高度：

```html
<span style="font-size:16px">abc</span>

<!-- 改成 -->

<div style="display:inline-block;font-size:16px;line-height:normal;">
  <span>abc</span>
</div>
```

测试字体16px的font metric高度是22px，改变line-height观察结果。

对于文字来说，他们占据的高度由line-height决定，一旦这个盒子超出了原本line box的高度，这个line box就需要扩大以容纳这个盒子。

### 4. vertical-align

