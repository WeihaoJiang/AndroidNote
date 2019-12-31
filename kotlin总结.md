---
title: Kotlin开发学习
date: 2018-11-19
tags: [Kotlin]
categories: Kotlin
---

## 本篇文章的背景以及夙愿

- 1 增强kotlin开发水平
- 2 保持更博客的习惯

## 为什么要学习kotlin
（因为本人是Android开发，所以仅从Android角度来说）
- 1 简介：
大型项目迭代过程中，阅读代码+写新需求是必不可少的，不论是同事的代码还是自己的代码，因此代码的简洁程度就成了提升生产力的关键
- 2 安全可靠：
kotlin的设计是可以防止程序因为某些错误挂掉的，举一个例子：空安全，简而言之就是kotlin帮助我们避开空指针异常这类错误，节省后期找错误一步一步debug的时间，这个后面详细说
- 3 互操作性：
和Java无缝连接，几乎可以使用所有Java库

## 个人体验kotlin与Java不同之处
（val,var常量变量声明，fun声明方法等等请自行百度，此处仅记录用惯了Java原生开发改用kotlin导致不舒服的地方）
### when关键字的使用
```
when (status) {
STATUS_CAR_GOOD -> {
selectedView(tv_car_status_good, iv_car_status_good)
curCarStatus = getData()?.priceResultKey?.high
}
STATUS_CAR_COMM -> {
selectedView(tv_car_status_common, iv_car_status_common)
curCarStatus = getData()?.priceResultKey?.middle
}
STATUS_CAR_DIFF -> {
selectedView(tv_car_status_diff, iv_car_status_diff)
curCarStatus = getData()?.priceResultKey?.low
}
}
```
###  for循环的变形使用
```
for (itemData in golds) {
num ++
if (num > 3) {
return
 }
 val itemCtrl = AppraiserItemCtrl(mContext, golds)
 val itemView = itemCtrl.createView(ll_appraiser_list)
 itemView.layoutParams = mLayoutParams
 ll_appraiser_list.addView(itemView)
 itemCtrl.setData(itemData)
 itemCtrl.bindView()
}
```
或
```
for (i in 1..100){

}
```
注意kotlin的区间是闭合的，第二个值始终是区间的一部分
### object关键字在声明单例，接口时的使用
```
mContentListRecyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
override fun onScrollStateChanged(recyclerView: RecyclerView?, newState: Int) {
super.onScrollStateChanged(recyclerView, newState)
if (recyclerView == null) {
return
}
when (newState) {
RecyclerView.SCROLL_STATE_IDLE -> {
if (!recyclerView.canScrollVertically(-1)) {
appbar.setExpanded(true, true)
}

}
RecyclerView.SCROLL_STATE_DRAGGING -> {

}
RecyclerView.SCROLL_STATE_SETTLING -> {

}
}

}

```
###  伴生对象代替static
```
companion object {
const val TYPE_AROUND = 0
const val TYPE_REFERENCE = 1
const val TYPE_RECOM_CAR = 2
}
```
companion object修饰的对象或方法，等同于静态对象或方法
### 创建单例
```
object Resource {
    val name = "Name"
}
```
### 压缩代码方面
- 1 findviewbyid
kotlin是不需要写findviewbyid的，可直接指向id
- 2 with和apply

### 可空性
- 1 ?.
可空修饰符
对有可能为空的变量用该修饰符修饰可以避免空指针异常，被？修饰的变量只会调用非空值的方法，若为空会返回null
- 2 ?:
kotlin中没有像Java中的三元判断，但个人感觉这个很像，一定程度上可以替代，详见代码
```
    fun strLenSafe(s:String):Int = s?.length ?: 0 
    >>>println(strLenSafe("abc"))
    3
    >>>println(strLenSafe(null))
    0
```
- 3 let
.let{ }是不为空的时候执行,否则什么都不会发生
```
val data = ...

data?.let {
    ... // 如果不为空，请执行此块操作
}
```
- 4 lateinit
在Kotlin中，声明为具有非空类型的属性必须在构造函数中初始化，但是往往不希望在构造函数中初始化，例如在通过依赖注入或单元测试的设置方法来初始化属性的时候，不能在构造器中提供一个非空的初始化语句，为了处理这种情况，就要在属性上加lateinit关键字来延迟初始化
```
class BuyAndSellAppraiseCtrl(context: Context) : Ctrl<GAppraiseResultResponse>(context) {
    private lateinit var tv_car_status_top_msg: TextView
    private lateinit var tv_ge_ren_che: TextView
    private lateinit var tv_ge_ren_che_value: TextView
    private lateinit var tv_ge_ren_che_area1: TextView
    private lateinit var tv_che_shang_che: TextView
    private lateinit var tv_che_shang_che_value: TextView
    private lateinit var tv_che_shang_che_area1: TextView
    private lateinit var tv_che_shang_shou_che: TextView
    private lateinit var tv_che_shang_shou_che_value: TextView
    private lateinit var tv_che_shang_shou_che_area1: TextView
    ...
}
```
lateinit只能够在var类型的属性中，不能用于构造函数，而且属性不能有自定义的getter和setting，这些属性必须是非空类型，并且不能是基本类型。
- 5 !!
不可空修饰符
个人认为使用场景不是很多，而且会带来很多问题，慎用
未完待续^_^

