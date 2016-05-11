# ReactNative性能调优

自从ReactNative出世,虽然官方一直尽可能的优化其性能，为了能让其媲美原生App的速度，但是现实感觉有点不尽人意。接下来介绍下实践中遇到的一些性能问题以及优化方案。注意以下对性能参数的依据是来自于React-Native自带的Perf Monitor.

### 1.Navigator页面切换动画优化
**场景:** 在Navigator还没出来时，导航器是由NavigatorIOS来实现的，当时觉得页面切换动画很流畅，但是一旦用Navigator后，发现定义的切换动画会使JS线程出现严重的掉帧(卡顿现象)。

**原因:** NavigatorIOS的切换动画是跑在UI主线程上，而不是JS线程上的，所以不受JS线程上的掉帧影响。当然官方还是推荐使用Navigator,具体原因请见官网的说明:
<http://reactnative.cn/docs/0.23/navigator-comparison.html>

**优化切换动画卡顿的问题：**

1. 使用API InteractionManager，它的作用就是可以使本来JS的一些操作在动画完成之后执行，这样就可确保动画的流程性。当然这是在延迟执行为代价上来获得帧数的提高。
2. 使用LayoutAnimation API(一次性动画)，在对动画中途无取消要求或者其他中途回调要求的(比如局部组件特定显示隐藏动画等)，则可以使用这个方案。

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

**优化问题:** 减少首屏加载的数据，实现数据懒加载，其先加载3页的数据量，然后在滑动的时候后台去取后面的数据(例如绑定到Slider组件的onMomentumScrollEnd事件中,每次取3条数据),最后每次保持6分钟的数据在组件中，其他数据则可放到localstorage中作为缓存。这样就可以减少首屏加载事件和提高用户体验

###4.组件响应速度的优化
**场景:** 一个页面包含多个类别的列表，由于列表都比较长，所以需要增加折叠功能并增加折叠动画，折叠按钮使用的是TouchableHighlight组件。问题是当我点击折叠或者展开按钮时出现延迟响应和动画掉帧的问题。

**原因:** 在TouchableHighlight组件的onPress方法中执行了setState的操作，由于列表的对象相对来说比较复杂需要大量计算的工作，所以导致了延迟响应和JS线程的掉帧。

**优化问题:** 使用requestAnimationFrame(fn)在下一帧就立即执行回调，这样就可以异步来提高组件的响应速度。而折叠动画则可以使用LayoutAnimation一次性动画来完成，保证其流畅性。

```
OnPress() {
  this.requestAnimationFrame(() => {
    \\封装之前OnPress中的操作，比如setState等。
  });
}
```
还有要提醒的是尽量优化组件的View结构，当View的层级很深时渲染的速度也会变慢
