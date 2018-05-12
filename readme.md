# 使用 sessionStorage 实现返回上一页浏览的历史位置

> 移动端列表页一般会采用滚动加载的方式，当点击列表中某一项进入到其它页面，再返回列表页的时候，会回到第一页，用户体验不够友好。
> 但移动端又不像 PC 端可以打开新页面，所以在这个需求点上，我使用了 sessionStorage 来实现了这个功能。

## DEMO

![](https://i.loli.net/2018/05/11/5af5692d2b088.png)

## 实现思路

思路其实很简单，就是在用户每次加载页面数据的时候，将本次加载的列表 json 数据、当前页码保存到 sessionStorage 里，需要注意的是，列表 json 数据是需要累加的，假设用户滚动加载到第三页，则 sessionStorage 里要存放三页的列表数据。

然后在用户点击列表某一项的时候，将当前列表距离页面顶部的距离记录下来，也存放到 sessionStorage 里即可。当然也可以给列表页增加一个滚动监听的事件，做到实时监听位置。

当用户从其它页面返回到列表的时候，先判断 sessionStorage 里是否有数据，如果有，则直接将 sessionStorage 里的列表数据复原到页面上，当前页码也更新成 sessionStorage 里存放的页码，这样做的目的是实现用户滚动页面可以继续往后加载数据。最后通过 sessionStorage 里存放的页面位置，定位到之前的位置即可。

如果返回到列表页，没有 sessionStorage 数据，那就什么都不做，还是按原有功能实现即可。

> 为什么不用 cookie 或者 localStorage 来实现？因为 sessionStorage 的特性最适合，关闭页面后自动失效。这样就不会出现用户下次打开列表页，还是显示之前的内容。

## 难点

实现思路清晰后，核心部分反而没有什么难点，倒是在配合 [MeScroll](http://www.mescroll.com) 使用上，发现了几个需要注意的地方。

> 不同的滚动加载控件需要根据控件本身一些功能设置实现我们想达到的效果

### 设置当前页码

通过 `mescroll.setPageNum(num);` 可以设置当前 page.num 的值，但通过阅读源码发现：

```javascript
/*设置page.num的值*/
MeScroll.prototype.setPageNum = function(num) {
    this.optUp.page.num = num - 1;
}
```

设置的页面会减 1 ，所以在触发 sessionStorage 进行设置页码的时候，需要手动加 1 。

### 隐藏上拉加载状态

在 demo 中我是使用 `mescroll.endBySize(dataSize, totalSize, systime);` 进行隐藏上拉加载的状态，通过阅读源码发现：

```javascript
/*联网回调成功,结束下拉刷新和上拉加载
 * dataSize: 当前页的数据量(必传)
 * totalSize: 列表所有数据总数量(必传)
 * systime: 服务器时间 (可空)
 */
MeScroll.prototype.endBySize = function(dataSize, totalSize, systime) {
    var hasNext;
    if(this.optUp.use && totalSize != null) {
        var loadSize = (this.optUp.page.num - 1) * this.optUp.page.size + dataSize; //已加载的数据总数
        console.log(loadSize, totalSize);
        hasNext = loadSize < totalSize; //是否还有下一页
    }
    this.endSuccess(dataSize, hasNext, systime);
}
```

这个方法实际上是通过 `( 当前页码 - 1 ) * 每页数量 + 当前页数量` 计算出已加载数据总数，然后去对比列表数据总数，判断出是否还有下一页。

既然理解了这个方法，那只要在复原 sessionStorage 里的列表数据后，模拟触发一次即可。

但这个方法里第一个参数 dataSize 并不好模拟，因为 sessionStorage 里只存放了列表数据合集。我的做法是通过 `数据合集数量 % 每页数量` 取出余数，这个余数其实就是我最后一次加载的列表数量，但需要注意的是，这个余数可能是 0 ，余数是 0 则代表最后一次加载的列表数量和每页数量一样，所以如果余数是 0 的时候，每页数量就是最后一次加载的列表数量。

> 每页数量就是 page.size 的值，默认是 10