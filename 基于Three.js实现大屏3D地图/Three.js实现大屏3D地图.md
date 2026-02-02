## 依赖安装

```json
yarn add three
yarn add @types/three
yarn add d3-geo
```

`three`库安装后在`node_modules`下其还包含核心`three/src`和插件`three/example/jsm`的源码，在开发调试时可以直接查阅。使用Three.js过程中会涉及到许多的类、方法及参数配置，所以建议安装`@types/three`库；不仅能提供类型提示，还有助于加快理解Three.js中的众多概念及关联关系。

`d3-geo`是`d3`库中独立出来专门用于处理地理数据可视化的模块。我们需要使用`d3-geo`中的部分方法来对原始的经纬度数据做[墨卡托投影](https://zh.wikipedia.org/w/index.php?title=%E9%BA%A5%E5%8D%A1%E6%89%98%E6%8A%95%E5%BD%B1%E6%B3%95&oldformat=true&variant=zh-cn)以在二维平面上正确定位。



## 搭建基础场景

