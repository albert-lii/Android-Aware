
# ListView 的 重用机制简介

## 一、RecycleBin
`RecycleBin` 是 `ListView`的重用机制的核心，它是 `ListView` 的一个内部类。
其中，有两个重要的变量我们要记住：  
- `mActiveViews`：缓存屏幕上可见的 itemView
- `mScrapViews`：缓存废弃的 itemView

**大致原理是：**   
1. `ListView` 的 item 滑出屏幕后，会从 ListView 上移除放入 `RecycleBin` 中
2. `ListView` 在移除一个 item 后，势必会再加载一个 item，这时会先从 `RecycleBin` 中寻找是否有相      同类型的 item 存在
3. 如果存在，则从 `RecycleBin` 中获取 item，加载到 `ListView` 中（`Adapter` 中的  `getView()` 方法中 `convertView` 不为 null 时，即为 `RecycleBin` 返回的 view）
4. 如果不存在，则重新创建一个 item（在 `Adapter` 中的 `getView()` 方法中 inflater 一个 view）

## 二、ListView 的 onLayout()
只要是 View，执行流程逃不过三个步骤：`onMeasure()`（测量 view 的大小）、`onLayout()`（计算 view 的位置）、`onDraw()`（绘制 view）；然而不知道什么原因，view 会执行两次 `onMeasure()` 和 `onLayout()` 方法。

而 `ListView` 本质上也是继承于 View，也需要走这三个步骤。`onMeasure()` 对于 `ListView` 没有什么特殊的，`onDraw()` 对于 `ListView` 也没有什么意义（子 view 负责绘制，`ListView` 本身不负责绘制），`ListView` 的重用机制核心主要集中在`onLayout()`  方法中。

### 1. 第一次 onLayout
- `ListView` 刚开始还没有任何 item，所以 `getChildCount()` 返回 0  
- 这时 `RecycleBin` 会调用 `fillActiveViews()` 方法缓存 item，因为没有 item，所以没有意义  
- 后期还会从 `mActiveViews` 中取出 item，因为 `mActiveViews` 为空，所以也不会取出任何 item  
- 最后 item 都是在 `Adapter` 中新创建的，然后添加到 `ListView` 中，直至塞满 `ListView` 控件为止，较耗时

### 2. 第二次 onLayout
- 因为第一次 `onLayout` 时已经添加了 item，所以第二次 `RecycleBin` 调用 `fillActiveViews()` 方法缓存 item 时就有值了，`mActiveViews` 不会空  
- 在后续方法中，会有一个 `detachAllViewsFromParent()` 方法移除 ListView 中的所有 item（如果不这么做，后续会再次添加 item，就重复加载数据了）。这时之前存储在 `mActiveViews` 中的 item 会被取出，避免重新创建 item。这里值得一说的是，`mActiveViews` 中被取出的 item 的位置会被置为 null，这意味着 `mActiveViews` 只能被使用一次，主要作用是缓存第一次 `onLayou`t 时创建的 item

## 三、ListView的滑动
以上主要是介绍了 `RecycleBin` 中的 `mActiveViews`，主要作用为缓存第一次 `onLayout` 时创建的 item，用于第二次 `onLayout` 时复用。下面介绍下 `mScrapViews` 的作用。

- item 完全滑出屏幕时，会被从 ListView 中移除，放入到 RecycleBin 中的 mScrapViews 中
- `ListView` 会从 `mScrapViews` 中寻找是否存在与即将加载的 item 相同类型的 view，如果存在则从 `mScrapViews` 中取出 view ，赋给 Adapter 中 `getView()` 方法中的 `convertView`，如果不存在，则 `Adapter` 中  `getView()` 方法中的 `convertView` 为 null，只能重新创建一个 itemView。
