# ReactNative性能调优

自从ReactNative出世,虽然官方一直尽可能的优化其性能，为了能让其媲美原生App的速度，但是现实感觉有点不尽人意。接下来介绍下实践中遇到的一些性能问题以及优化方案。以下对性能参数的依据是来自于React-Native自带的FPS Monitor.



### 1.Navigator页面切换动画优化
**场景:** 在Navigator还没出来时，导航器是由NavigatorIOS来实现的，当时觉得页面切换动画很流畅，但是一旦用Navigator后，发现定义的切换动画会使JS线程出现严重的掉帧(卡顿现象)。

**原因:** NavigatorIOS的切换动画是跑在UI主线程上，而不是JS线程上的，所以不受JS线程上的掉帧影响。当然官方还是推荐使用Navigator,其原因如下：

<1.Navigator扩展性的API设计使得它完全可以通过js定制，而NavigatorIOS则无js层面的定制；

<2.Navigator使用JavaScript编写，iOS和Android都可以使用，而NavigatorIOS只能在IOS上

<3.Navigator优化后的动画效果还算不错，而且官方还在不断改进中，当然这个动画比不上NavigatorIOS那么顺滑。但NavigatorIOS并不在FaceBook的应用中使用，也不是其主导开发，而是开源社区主导开发。所以可能坑多又没人给出填坑的解决方法。

所以我们选择导航的时候尽量选择Navigator吧。

**优化切换动画卡顿的问题：**

1. 使用API InteractionManager，它的作用就是可以使本来JS的一些操作在动画完成之后执行，这样就可确保动画的流程性。当然这是在延迟执行为代价上来获得帧数的提高。

```
InteractionManager.runAfterInteractions(() => {
  // ...耗时较长的同步的任务...
         //更新state也需要时间
      that.setState({
       ...
             });
      //获得某些数据，需要较长时间等待        
      that.getData(arguments);
    });

  },
});
```

2. 使用LayoutAnimation API(一次性动画)，在对动画中途无取消要求或者其他中途回调要求的(比如局部组件特定显示隐藏动画等)，则可以使用这个方案。我们可以在调用setState之前，调用LayoutAnimation方法。代码如下：

  ```
  ...
      animations: {
        layout: {
             //easeInEaseOut的config		
            easeInEaseOut: {
             //duration 动画持续时间，单位是毫秒
                duration: 300,
             //create, 配置创建新视图时的动画。
                create: {
                    type: LayoutAnimation.Types.easeInEaseOut,
                    property: LayoutAnimation.Properties.scaleXY,
                },
             //update, 配置被更新的视图的动画。
                update: {
                    delay: 100,
                    type: LayoutAnimation.Types.easeInEaseOut,
                }
            }
        }
    };
  ...
  //计划下一次布局要发生的动画
  LayoutAnimation.configureNext(this.animations.layout.easeInEaseOut);

  ```


###2.数据类型的优化
**场景**：基本上每个页面都需要加载和渲染数据，如果页面列表数据结构复杂,有时刷新数据时state中的未必有修改，但是遇到这样的语句this.setState({data:samedata}) ,界面却被重新render.

**原因:** 这是react-native的生命周期，当你调用setState时，总是会触发render的方法

**优化数据问题：**可以使用shouldComponentUpdate生命周期方法，此方法作用是在props或者state改变且接收到新的值时，则在要render方法之前调用。此方法在初始化渲染的时候不会调用，在使用 forceUpdate 方法的时候也不会。所以在这个方法中我们可以增加些判断规则来避免当state或者props没有改变时所造成的重新render.

```
shouldComponentUpdate: function(nextProps, nextState) {
  return nextProps.value !== this.props.value;
}

```

但仅仅做这层判断是不够的，如果是一个列表的对象，例如下面的例子：

```
var Component = React.createClass({
    getInitialState() {
        return {
			...
            data:{
                valueobj: {
                    v1: 'v1',
                    v2: 'v2'
                }
            }
        }
    },
  shouldComponentUpdate(nextProps, nextState){
        return (
         return nextProps.data !==this.props.data; );
    }
```
这里即使使用了shouldComponentUpdate中的判断，但却一直返回true，导致还会执行render。所以必须对对象所有的键值进行进行比较才能确认是否相等。这里推荐使用facebook自家的immutablejs。一个不可变数据类型的库。使用后可以直接使用一下的写法达到我们之前的目的(即使是对象都可以完美的做比较)。修改后代码如下：

```
var { List, Map } = Immutable;
var Component = React.createClass({
    getInitialState() {
        return {
			...
            data: Immutable.fromJS({
                 valueobj: {
                    v1: 'v1',
                    v2: 'v2'
                }
            })
        }
    },
  shouldComponentUpdate(nextProps, nextState){
        return (
         return nextProps.data !==this.props.data; );
    }
...
```

immutablejs其他的具体用法请见:<http://facebook.github.io/immutable-js/>

###3.数据加载的优化
**场景:** 在首屏页面加载时，加载前6分钟的数据分6页显示，并需保持当前选择页的时间的前6分钟，如果按照此场景开发所遇到问题是：首页加载时间太长，加载新数据时页面显示加载用户体验不顺畅

**原因:** 首页请求数据量过大,导致首屏页面加载很慢;后台数据更新时导致用户体验不顺畅

**优化问题:** 减少首屏加载的数据，实现数据懒加载，其先加载3页的数据量，然后在滑动的时候后台去取后面的数据(例如绑定到Slider组件的onMomentumScrollEnd事件中,每次取3条数据),最后每次保持6分钟的数据在组件中，其他数据则可放到localstorage中作为缓存。这样就可以减少首屏加载事件和提高用户体验。加载数据的滚动列表示例代码如下：

初始化(定义数据data)：

```
...
getInitialState: function(){
        return{
      title: title,
      data: data,
		...
    };
  },
```  

滚动列表的事件：分为左滑每次到3的倍数页面取当前取过的数据的前3分钟的历史数据；右滑则取之后的时间。

```  
  ...
  //滚动列表的事件
  momentumScrollEnd: function(e, state){
    var data = this.state.data;
    var that = this;
    //当无数据可更新时禁止滚动
    if(!data){
      return 0;
    }
	...
    //（1）左移，手势<－
    //时间倒退，index为滑动列表的序列号。每滑动到3的倍数的页面则去加载历史数据
    if(index % 3 === 0 && currentIndex < index){
      hour = lastHour;
      lastMin = lastMin - 1;
      var newDate = new Date(year, month, dayri, hour, lastMin);
      //加载历史数据
      this.getNewData(newDate, newDate.getMinutes(),3, 			function(cbData){
        		for(var i in cbData){
          			data.push(cbData[i]);
        		}
        		that.setState({
          			data: data
        		});
      });
    }
    //(2)右移，手势－>
        //同时监测移动的方向，采用touch事件
    //判断4个事件才能确保滑动的方向（touchStart、touchEnd、scrollBeginDrag、momentumScrollEnd
    //50为阈值控制
    var disLoc = parseFloat(this.state.touchEndLocX) - parseFloat(this.state.touchStartLocX);
    var disPage = parseFloat(this.state.touchEndPagex) - parseFloat(this.state.touchStartPagex);
    if(this.state.isLoadFront && state.offset.x === 0 && disLoc > 50 && disPage > 50){
      hour = firstHour;
      firstMin = firstMin + 1;
      var newDate = new Date(year, month, dayri, hour, firstMin);
      ...
      //右移，加载当前时间的数据
        this.getNewData(newDate, newDate.getMinutes(), 1, function(cbData){
          for(var j = cbData.length - 1; j >= 0; j--){
            data.unshift(cbData[j]);
          }
 		...
          });
        });
      }else{
        alert('当前数据为最新数据，暂不用更新');
      }
    }
	...
  }
  ...
```






###4.组件响应速度的优化
**场景:** 一个页面包含多个类别的列表，由于列表都比较长，所以需要增加折叠功能并增加折叠动画，折叠按钮使用的是TouchableHighlight组件。问题是当我点击折叠或者展开按钮时出现延迟响应和动画掉帧的问题。

**原因:** 在TouchableHighlight组件的onPress方法中执行了setState的操作，由于列表的对象相对来说比较复杂需要大量计算的工作，所以导致了延迟响应和JS线程的掉帧。

**优化问题:** 使用requestAnimationFrame(fn)在下一帧就立即执行回调，这样就可以异步来提高组件的响应速度。而折叠动画则可以使用LayoutAnimation一次性动画来完成，保证其流畅性。而对于某些状态更新，setNativeProps方法可以让我们直接修改原生视图组件的属性，而不用通过setState来重新渲染结构，这样能使整个组件响应速度变快。


```
OnPress() {
  this.requestAnimationFrame(() => {
    \\封装之前OnPress中的操作，比如setState等。
  });
}
```
还有要提醒的是尽量优化组件的View结构，当View的层级很深时渲染的速度也会变慢


###5.资源优化
**场景:** 这里说的资源包括reactnative打出来的Bundle，图片等静态资源。react-native的一股脑儿的打包方式，无疑一下子增大了Bundle包大小。还有一个页面多多少少会包含一些图片，特别是在一些商业APP中，图片是对内容一种补充，能让提高用户体验。为了能更快的加载图片，可以把图片打入包中，当然这个代价是巨大的。对APP来说，控制包的Size不管从商业方面还是开发性能方面都是一个很重要的参数。

**优化问题:**

1. 对于Bundle过大，我们可以通过一些思路来优化它，首先是对其尝试进行拆包，然后对拆包进行约束，使公共基础那部分拆成一类包，使其可以按需加载本地文件，而像业务逻辑等则拆成另一类包，使其可以按需加载线上文件。
2. 图片我们可以对其转成webp格式。webp大家应该都很熟悉了,它既支持有损压缩又支持无损压缩的图片文件格式。根据官方介绍其无损压缩后的WebP比PNG文件少了45％的文件大小，即使PNG文件经过压缩工具压缩之后，WebP 还可以减少28％的文件大小，这可以大大提高移动端的图片加载速度。据官网介绍在IOS平台中,每次调整Image组件的大小，都需要重新裁剪和缩放原始图片，重新渲染界面。这个操作开销会非常大，尤其是大的图片。比起直接修改尺寸，更好的方案是使用transform: [{scale}]的样式属性来改变尺寸。比如当你点击一个图片，要将它放大到全屏的时候，就可以使用这个属性。


###6.页面加载与显示优化
**场景:** 某些页面需要访问一个或多个业务数据服务，虽然取数据是异步，但是页面总是会有一段较长的loading的时间。

**优化问题:** 对于首屏所需的数据服务的访问，使其在页面加载阶段尽早的发起数据请求，这样有助于减少等待数据的时间。而对长时间的Loading可能会降低用户体验的问题，我们可以使用Fake页来提高用户体验。先显示一个Fake页，等数据请求后并执行了相应的业务逻辑后，再替换成真正的页面。


以上是我们在实践中的一些优化心得，优化之路漫漫，吾将上下而求索。特别是在React-Native还在成长期这个阶段，优化变得尤为重要。期望React-Native未来在性能上有更好的突破。
