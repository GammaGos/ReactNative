### React Native Bundle 拆分的尝试

### 引言

React Native以其独到的特性，吸引着互联网公司纷纷为之投入或多或少的人力。在实际的开发过程中，开发者们也确实尝到了甜头，它的组件化思想、热更新机制以及jsx和es6等的引入，都给开发者们带来了很大的便利。也难怪在npm和github上，每天都会有很多react-native的新模块出现。这也充分表明了各大公司对其的看好。然而，从目前qq群、微信公众号、社区、论坛等各大信息交流平台中了解到，大家都是保持在研究和观望状态，顶多把某个不重要的页面交给React Native来练手。其中缘由纷繁复杂。今天我们这里主要是探讨——————bundle文件太大。

### 现状

React Native应用的开发者们，在项目开发完后，都会遇到一个问题，生成的bundle文件太大。一个AwesomeProject项目，在没有什么逻辑代码的情况下，打完之后约530k。随着业务的增多，业务复杂性的上升，文件的大小势必会急剧增大。react-native打包成一个bundle的做法，必定是要得到解决的。

### 分析


react-native默认提供的打包方式有两种：

+	离线打包

```
react-native bundle
	--entry-file index.ios.js
  	--platform ios
  	--dev true
  	--bundle-output dest/main.jsbundle
  	--assets-dest dest

```


+	在线打包

```

http://localhost:8081/index.ios.bundle?platform=ios&dev=true

```
具体有哪些参数可以打开如下文件进行查看：

```
$youProjectRoot/AwesomeProject/node_modules/react-native/local-cli/bundle/bundleCommandLineArgs.js

如：
module.exports = [
  {
    command: 'entry-file',
    description: 'Path to the root JS file, either absolute or relative to JS root',
    type: 'string',
    required: true,
  },
  ......

```
官网中还给出了一些其它的使用方式，地址：

```
https://github.com/facebook/react-native/tree/master/packager

```

不过，不论哪种方式都是只有一个“entry-file”，然后根据“entry-file”去进行依赖分析、文件压缩等操作，最后输出在“bundle-output”中。然后通过NSBundle的URLForResource方法来指定加载打好的的bundle文件。

如：

```
jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
```


这样的打包模式，对用户体验来说是非常不错的。但是考虑到国内的网络状况以及对App size的控制，打成一个Bundle的模式在国内还是行不通的。


### 思考

在传统的Hybrid开发中，要解决文件太大的问题，我们常常会想到如下几个办法：

+	进行拆包
+	按需加载本地文件
+	按需加载线上文件

那么，能否把Hybrid开发中的经验应用在React Native项目中呢。在React Native项目中，针对文件大的问题，我们做了如下尝试：

+	多业务进行拆包

借助gulp、grunt等工具，通过配置不同的任务，在调用React Native提供的打包命令，可以将App打包成多个文件。

+	按需加载本地文件

在开发环境的情况下，React Native是支持加载本地文件的。这里想要做的是，在打包完的bundle中也可以加载本地文件，这就需要对require进行扩张了。

+	按需加载线上文件

在开发Hybrid时，为了减少包体积。开发者们经常会将一些不重要的页面或文件，走线上动态获取的方式。这个功能只有在web端的requirejs中有，React Native和webpack中都是不支持的。要实现此项功能，需要对React Native中的require进行扩张。

+	按需加载react-native模块

不论是reactjs还是react-native，在代码的组织方式上都是按模块进行划分的。可能Facebook也意识到react框架太大了。这个模块划分的方式，给开发者们的按需载入创造了机会。

### 实现

这里简单阐述下部分功能的实现思路

+	React Native自身模块拆分

在打完的main.jsbundle中，常常会看到好多polyfills的文件，那这些文件从哪里来的呢。打开

```
node_modules/react-native/packager/react-packager/src/Resolver/index.js
```
 文件，会看到这些polyfills文件都是在这里设置的，

 ```
path.join(__dirname, 'polyfills/String.prototype.es6.js'),
path.join(__dirname, 'polyfills/Array.prototype.es6.js'),
......

 ```
由名字可以看出，这些是用来对es6、es7进行适配的。所以代码中如果只有es5的语法是不是就可以不需要这些文件了呢，这也是个优化点，不过看起来量不大。

有人可能经常会有这样的想法：我们实际项目中用到的React Native模块其实并没几个，我们在打包的时候，是否可以只打包我们需要的模块呢。我们找到文件


```
/node_modules/react-native/Libraries/react-native/react-native.js

```

可以看到所有React Native的模块定义都是在这里了，包括Components、APIs等等。

```
var ReactNative = {
  // Components
  get ActivityIndicatorIOS() { return require('ActivityIndicatorIOS'); },
  get ART() { return require('ReactNativeART'); },
  ......

```
所以，可以在打包的时候，根据实际情况，通过脚本等手段，注释掉一些用不到或不常用的模块以减少输出的体积。当然也可以把部分不常用的模块，抽出来单独作为一个文件，在需要的时候，通过按需加载的方式引入进来。

+	业务模块拆分

App的设计一般都是按照业务线划分的。每个业务都对应一套自己的逻辑。当然也有部分业务线会出现依赖情况。按React Native提供的打包方法，将所有业务线的逻辑都打在一起，势必会造成好多业务线代码的浪费。有可能那个业务线就根本不会被用户访问到。所以我们就想着能不能将一些基础的、公共的业务线打在一起，其它独立的业务线都各自独立成包。

React Native提供的打包方法允许输入一个入口文件，那么这个入口文件可以是整个App的入口，也可以是各业务线自己的入口。由此我们可以将各业务线单独成包，但这样的结果并不能直接投入使用。可以想到，这里并没有过滤机制，各业务线之间一些模块会被重复的打进去也包括react-native模块。而React Native打包提供的参数中也只有blacklist会涉及一些过滤，但却无法满足我们的需求。

还好packager为我们提供了很多可以的API。通过参考

```
/node_modules/react-native/local-cli/bundle/buildBundle.js
```
中的打包逻辑，我们以一个入口文件的打包为例，可以将打包逻辑设计成如下：

```
1、加载打包需要用到的模块
var config = require('config.js')
var ReactPackager = require('react-native/packager/react-packager')
var Bundle = require('react-native/packager/react-packager/src/Bundler/Bundle')
var saveAssets = require('react-native/local-cli/bundle/saveAssets')
var outputBundle = require('react-native/local-cli/bundle/output/bundle')

2、创建client
var client = ReactPackager.createClientFor({
    projectRoots: config.projectRoots,
    blacklistRE: config.blacklistRE,
    ...
})
3、调用outputBundle进行打包，将打包后的bundle返回
outputBundle.build(client, {
  entryFile: config.file,
  ......
})
4、分析bundle中的module，将符合条件的module加入到新的bundle中
定义一个新的bundle
var newBundle = new Bundle();
bundle.getModules().forEach(function (module) {
	if(filter(module.sourcePath)){
		newBundle._modules.push(module);
	}
	......
})

5、定义过滤机制
function filter(path){
	var ret = true;
	if(
	(path.indexOf('/react-native/')!=-1)||
	(sourcePath.indexOf('/fbjs/')!=-1)||
	......
	){
		ret = false;
	}
	return ret;

}
上只是个简单的过滤，在复杂的过滤中，还需要调用ReactPackager.getDependencies找到每个模块的依赖，然后在过滤的时候调用过滤模块的依赖，依次递归才能达到真正的滤掉。

6、对module进行合并、替换等处理
newBundle.finalize()

7、调用outputBundle输出新的bundle

outputBundle.save(newBundle, {}, false)         

```
到此，一个带有过滤功能的打包脚本就基本成型了，之后的多文件入口同时打包的功能，也就是要在上面做些扩扩展就可以了。

在打包方面，其实也也可走网络打包，packager的网络打包逻辑中，凡是请求以.bundle结尾的文件，都会对这个文件进行打包。而其它格式的文件，则请求什么返回什么。所以可以根据该特性来实现打包。可以将过滤条件作为querystring的方式传递过去，然后在

```
react-native/packager/react-packager/src/Bundler/index.js

```
文件中对querystring进行拦截，并实现其过滤功能。

然而在实际的拆包中会发现，packager中打出的包都会将模块名称替换为数字id。如：

```
__d(14,function(s,t,i,o){"use strict";var r={OS:"ios"};i.exports=r});

__d(30,function(n,t,o,r){"use strict";var u,e=t(31);u=e.now?function(){return e.now()}:function(){return Date.now()},o.exports=u});


```
这导致拆出的包中，引入不到某些模块，因为不是在一起打包，模块的id都对不上，或者会出现重复的情况。

我们的思路是打包的时候不进行id的替换，依然使用原有的模块名称，做到类似在web中requireJS使用的那样。
找到文件

```
node_modules/react-native/packager/reat-packager/src/Resolver/index.js

将如下代码中的moduleName，替换为model的绝对路径
function defineModuleCode(moduleName, code, verboseName = '') {
  return [
    `__d(`,
    `${JSON.stringify(moduleName)}/*<-替换的地方*/ /* ${verboseName} */, `,
    `function(global, require, module, exports) {`,
      `${code}`,
    '\n});',
  ].join('');
}

```
这样只完成了define（如：define（0,...））中的名称替换，我们还需要找到require（如：require(0)）中的名称替换，于是找到如下文件

```
node_modules/react-native/packager/reat-packager/src/Bundle/Bundle.js

在super（BundleBase）中，定义一个获取模块的方法getModuleName，将下面的super.getMainModuleId替换为super.getModuleName，这样在_addRequireCall就可以拿到模块的绝对路径了

_addRequireCall(moduleId) {
    const code = `;require(${JSON.stringify(moduleId)});`;
    const name = 'require-' + moduleId;
    ......
}

finalize(options) {
    options = options || {};
    if (options.runMainModule) {
      options.runBeforeMainModule.forEach(this._addRequireCall, this);
      this._addRequireCall(super.getMainModuleId());/*<-替换的地方*/
    }

    super.finalize();
  }

```

这样就完成了模块名称的保留，我们就可以愉快的使用我们的拆包模块了。


+	按需加载实现

经过上面的介绍，我们已经完成了模块的拆分。那么光有独立的模块还是不能让App运行起来，需要有一种能力将这些模块联系起来，这就是模块加载机制。常规的加载会有如下两种场景：

1、本地模块

有时候为了加快页面打开速度，我们常常会选择将首页和非首页的页面进行分开打包，在App启动时，只加载首页的模块，待首页模块加载完毕后，再去异步的加载后续页面的模块。这里的本地模块加载就是用在这种场景中。那么在React Native中该如何实现这种加载方式呢。要读写本地文件，光有javascript是办不到的，所以一定要借助native的能力。简单的代码实现如下：

```
#import "RCTBridgeModule.h"
@implementation RequireLocal
RCT_EXPORT_MODULE()
RCT_EXPORT_METHOD(loadPath:(NSString *)path callback:(RCTResponseSenderBlock)callback)
{
  NSString *filePath = [[NSBundle mainBundle] pathForResource:path ofType:nil];
  if ([[NSFileManager defaultManager] fileExistsAtPath:filePath]) {
    NSString *content = [[NSString alloc] initWithContentsOfFile:filePath
    ......
}
@end
```
代码的流程为：按照React Native中对native模块封装的规范，实现RCTBridgeModule协议，并通过定义宏RCT_EXPORT_MODULE、RCT_EXPORT_METHOD将native模块的功能暴露给javascript来调用。在native的模块中，采用NSBundle的pathForResource方法，将文件路拿到。再借助NSString的initWithContentsOfFile方法获取到文件的内容。然后在javascript中，将拿到的内容，进行一次包装，如：

```
var str='__d("'+filePath +'", function(global, require, module, exports) {'+
	content+
'})'

```
最后调用eval，便可将拿到的内容执行到当前的jscontext中。

2、线上模块

在App的开发中经常会为了控制size大小而发愁，尤其是苹果的100m限制，所以各业务线都在绞尽脑汁的想办法减size。自然而然的大家就想到了将一些资源放在服务端，在需要的时候将其异步加载下来，也就是常常听说的直连。对于服务器异步加载的实现，代码如下：

```
fetch(filePath)
.then((response) => response.text())
.then((responseText) => {
......
```
代码的流程为：采用React Native提供的fetch方法，将需要的模块异步的从服务器上拉回来，接下来的动作，和上面的“本地模块”的逻辑一样。在实际的模块加载中，我们还需要对模块进行缓存，以提高模块的访问速度。


### 后续

在经过上面的介绍中，我们应该大概知道拆包和按需加载的实现原理。但是大家也都看到了，这要侵入react-native的代码中，进行很多地方的修改。这样不利于之后对react-native的版本升级。所以我们需要想一种更合理的解决办法。也就是我们现在正在做的一个尝试。将React Native中的cmd模块，在线下或运行时编译为AMD模块，然后调用r.js的来对其进行打包，以达到干净的完成拆包和按需加载的功能。而且r.js的打包配置的灵活度我觉得比packager、webpack、browserify等工具都灵活好使。
