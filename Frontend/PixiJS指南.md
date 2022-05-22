> 先前完成镜像计划时要选择一个`2D Canvas`框架，多方比对后，选择了基于`WebGL`的`PixiJS`。

# 介绍

官方网站对`PixiJS`介绍如下：

> PixiJS 的核心是一个渲染系统，它使用 WebGL（或可选的 Canvas）来显示图像和其他 2D 视觉内容。
>
> 它提供了完整的场景图，并提供交互支持以启用处理点击和触摸事件。
>
> 它是现代 HTML5 世界中 Flash 的自然替代品，但提供了更好的性能和像素级效果，超出了 Flash 所能达到的范围。
>
> 它非常适合在线游戏、教育内容、交互式广告、数据可视化……任何需要复杂图形的基于 Web 的应用程序。再加上 Cordova 和 Electron 等技术，PixiJS 应用程序可以作为移动和桌面应用程序分布在浏览器之外。

总结以下，`PixiJS`的优势：

1. 轻量，以渲染功能为核心。
2. 移植性，轻易在前端项目中引用。
3. 完整且最新的说明文档。

在本文章中，我们先来完成以下图片的内容。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9cb433e6ff24674a476b638fb0df758~tplv-k3u1fbpfcp-watermark.image?)

# 安装

- CDN 引入全局对象`PIXI`

```html
<script src="https://pixijs.download/release/pixi.js"></script>
```

- NPM 引入

```bash
npm install --save pixi.js
```

```js
import * as PIXI from "pixi.js";
```

# 基础知识

## APP

先在`Html`中添加对应的标签，用以存放画布内容

```html
<div class="pixiDom" ref="pixiDom"></div>
```

接着创建全局`app`对象，附加到页面对应标签中。

```js
let app = new PIXI.Application({ width: 640, height: 360 });

// 或 document.body.appendChild(app.view);
pixiDom.appendChild(app.view);
```

这里可以传入多个属性值，宽高为基本的属性，一些常用属性包括：
|属性|类型|描述|
|---|---|----|
|width|number|宽度|
|height|number|高度|
|view|HTMLCanvasElement|Canvas 元素|
|autoDensity|boolean|根据 css 像素 resize|
|backgroundColor|number(0x)|背景色|
|backgroundAlpha|number|透明度|
|powerPreference|string|"high-performance"设置高性能|
|...|||

下面介绍几类常用的`Pixi`对象，容器、图形、文字、精灵。

## Container

`Container`表示容器，在`Pixi`中使用的最为频繁，可以理解为`html`中的`div`标签，容器（包括其它各种对象）有着它们的位置`position`，枢纽`pivot`等常用属性对象，且能够通过`addChild`来添加子对象，通过`removeChild`来删除子对象，类似`DOM`的增删。

```js
let container = new PIXI.Container();
container.addChild(mask);

app.stage.addChild(container);
// app.stage.removeChild(container);
```

### position && pivot

一个容器的位置`position`是相对于其父容器的`(0,0)`来进行相对于本容器枢纽的位置`(0,0)`来定位，我们常常需要做出动态的居中效果，有以下两种写法：

```js
const setCenter = (son, parent) => {
  son.position.x = parent.width / 2 - son.width / 2;
  son.position.y = parent.height / 2 - son.height / 2;
};
```

这种方法类似于`Html`中的一种居中的策略，先移动父元素的 50%，再反向根据本元素的宽度（高度）移动 50%，如下：

```html
.parent { position: relative; height: 200px; widthL: 200px; } .son { position:
absolute; left: 50%; transform: translateX(-50%); }
```

但是以上`Pixi`实现居中的方法，在实现旋转或缩放效果时，会需要设置旋转中心`pivot`，而旋转中心又会影响其位置，因此下面的方法也是常用的居中策略。

```js
const setCenter = (son, parent) => {
  son.position.x = parent.width / 2;
  son.position.y = parent.height / 2;

  son.pivot.x = son.width / 2;
  son.pivot.y = son.height / 2;
};
```

在该居中策略中，实际上是枢纽`pivot`，或者说中心，距离父元素的左侧一半，和顶部一半。

## Graphics

`Graphics`表示图形，可以借助它来绘制矩形 Rect/圆形 Circle/椭圆 Ellipse/多边形 Polygon。

经常借助`Graphics`来为容器绘制`Container`的矩形遮罩：

```js
let mask = new PIXI.Graphics();

// Graphics.beginFill(color?: number, alpha?: number)
mask.beginFill(0xff387d);

// Graphics.lineStyle(width: number, color?: number, alpha?: number, alignment?: number, native?: boolean)
mask.lineStyle({ color: 0xffffff, width: 4, alignment: 0 });

// Graphics.drawPolygon(...path: number[] | PIXI.Point[]):
// Graphics.drawCircle(x: number, y: number, radius: number)
// Graphics.drawRect(x: number, y: number, width: number, height: number)
mask.drawRect(0, 0, 640, 700);

mask.endFill();

container.addChild(mask);
```

所有图形绘制前需要调用`beginFill`来确定填充颜色及透明度，在绘制结束后需要调用`endFill`来应用绘制。

在确定填充颜色后，可以指定画笔样式`lineStyle`来确定绘制的图形的边的样式，然后就可以调用`drawRect`等来绘制相应的图形了。

## Text

`Text`即表示文字。文字具备其相应的`CSS`属性，如下：

```js
let innerText = new PIXI.Text(text, {
  fontSize: 80,
  fill: 0xffffff,
  stroke: "#4a1850",
  strokeThickness: 3,
});
container.addChild(innerText);
```

### TextStyle

文字的样式包含以下几部分：字体，外观，阴影，布局（在官方文档中还包含着`**Utilities**`实用，此处不提）

- 字体包含着文字的大小，字体族等。

- 外观确定文字的填充颜色`fill`，轮廓`stroke`等。

- 阴影通过`dropShadow`定义，包括`dropShadowColor`,`dropShadowBlur`等。

- 布局主要影响文字的位置及间距等，通过`align`确定对齐方式，`wordWrap`和`wordWrapWidth`确定文字间隙等。

为了将文字样式保存，可以定义一个`TextStyle`，以做到在创建文本时，多次利用`Style`。

```js
// from Pixi examples
const style = new PIXI.TextStyle({
  fontFamily: "Arial",
  fontSize: 36,
  fontStyle: "italic",
  fontWeight: "bold",
  fill: ["#ffffff", "#00ff99"], // gradient
  stroke: "#4a1850",
  strokeThickness: 5,
  dropShadow: true,
  dropShadowColor: "#000000",
  dropShadowBlur: 4,
  dropShadowAngle: Math.PI / 6,
  dropShadowDistance: 6,
  wordWrap: true,
  wordWrapWidth: 440,
  lineJoin: "round",
});

const richText = new PIXI.Text(
  "Rich text with a lot of options and across multiple lines",
  style
);
```

### Button

基于以上已学到的知识，我们来完成一个按钮“组件”，由于按钮本身可以设置颜色，所以我们以`Graphics`为父容器，在其中嵌入`Text`子元素，可传入`TextStyle`样式。

这里涉及到点击事件，通过查阅，我们需要设置`button`的两个属性，`interactive`可交互，`buttonMode`按钮模式，接下来就可以用监听的方式添加`click`事件，如下：

```js
// let button = new PIXI.Graphics();
button.interactive = true;
button.buttonMode = true;
button.on("click", (event) => {
  clickHandler && clickHandler(event);
});
```

将以上因素结合，我们便可完成一个`newButton`组件，创建`Graphics`，增加点击事件，创建`Text`并添加到父元素中，最后将按钮

```js
const newButton = (text, style, clickHandler) => {
  let button = new PIXI.Graphics();
  button.beginFill(0xffffff); // 可传参定制
  button.drawRoundedRect(0, 0, 200, 50, 5); // 可传参定制
  button.endFill();

  button.interactive = true;
  button.buttonMode = true;
  button.on("click", (event) => {
    clickHandler && clickHandler(event);
  });

  let innerText = new PIXI.Text(text, style);
  button.addChild(innerText);
  setCenter(innerText, button);

  return button;
};
```

## Sprite

Sprite 称精灵，在各种动画框架中均可见到它的身影，可以通过图片来创建。

```js
let sprite = PIXI.Sprite.from("/imgs/E.png");

container.appendChild(sprite);
```

### Ticker

这里的`Ticker`，个人理解为类似`RequestAnimationFrame`的逐帧响应函数，每一帧将会触发一次相应函数。

```js
app.ticker.add((delta) => {
  sprite.rotation += 0.1;
});
```

根据以上便可实现`Sprite`的旋转效果。

### 雪碧图

通常为了实现动态效果，可以连续切换不同图片，`Pixi`支持雪碧图的方式。主要通过`PIXI.extras.AnimatedSprite`对象操作雪碧图，其传入的参数为纹理数组，通过`PIXI.utils.TextureCache`来切片图片，获取多个`frame`。

推荐查阅这篇文章：
[学习 PixiJS — 动画精灵 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903760561438728#heading-0)

# 总结

`Pixi`在使用上确实非常舒适，且很容易就嵌入到各种框架中。

在我的项目中使用`Pixi`绘制了三个页面，开始界面/介绍界面/主界面，均进行一定的封装，比如`setCenter`，`setRotate`，`newButton`等，这些封装在小游戏开发的过程中意义非凡，关于上述内容的基本实践，可以查看我的项目，代码在`/src/pixi/index.js`文件中。
[MrPluto0/arVision (github.com)](https://github.com/MrPluto0/arVision)

# 相关

[PixiJS: Getting Started](https://pixijs.io/guides/basics/getting-started.html)

[PixiJS API Documentation](https://pixijs.download/release/docs/PIXI.Application.html)

[MrPluto0/arVision (github.com)](https://github.com/MrPluto0/arVision)

[学习 PixiJS — 动画精灵 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903760561438728#heading-0)
