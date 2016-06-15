## moles-packer 开源啦。。。。。。


经过几天的连续奋战，moles-packer终于开源啦。感谢各位在moles-packer发布之初给予的鼓励和支持。有你们的鼓励我们才会有现在动力，我们是有信心将moles-packer做到功能最强大，使用最方便的React Natie打包、拆包工具。

我们将moles-packer开源到携程的公众账号ctripcorp上，上面还有很多有意义的开源项目，有种moles-packer找到组织的感觉,并将moles-packer升级到了0.1.3版本。

现在向诸位回报下目前的进展和计划

### 已完成

+	支持react、react-native打成common.jsbundle

+	支持除react、react-native以外的业务代码打成bu.jsbundle

### 待完成

+	单业务线拆包的支持

	+	common bundle的生成可配置化

	+	业务模块多bundle的支持

	+	load拆包和merge拆包iOS的支持

	+	load拆包和merge拆包Android的支持

+	多业务线拆包的支持

### 资讯


很高兴的告诉大家，我们的moles-packer即将推出0.2.0版本，该版本的API将会调整为更符合大家习惯的格式，如下：

```
input:"",
output:"",
paths:{
  "common":"./common.jsbundle",
  "foo/bar/bip":"foo/bar/bip/index.js",
  "foo/bar/bop":"foo/bar/bop/main.js",
  "foo/bar/bee":"foo/bar/bee/index.json",
  "react":"react",
  "react-natve":"react-native":
},
common:{
     name: "common",
     include: ["react","react-native"]
 },
business: [
  {
       name: "foo/bar/bip",
       exclude: [
           "foo/bar/bop"
       ]
   },
   {
        name: "foo/bar/bop",
        include: [
            "foo/bar/bee"
        ]
    }

 ],
dev:

```



### 相关的地址

[github地址](https://github.com/ctripcorp/moles-packer/)：https://github.com/ctripcorp/moles-packer/

[npm地址](https://www.npmjs.com/package/moles-packer)：https://www.npmjs.com/package/moles-packer

### 最后，大家如有什么想法或者建议欢迎发我们的TS组邮箱
<ctrip-moles@ctrip.com>
