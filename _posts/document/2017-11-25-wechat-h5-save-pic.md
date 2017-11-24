---
title:  "微信H5页保存当前页面为图片踩坑"
date:   2017-11-25
categories: 文档
tag: 微信 H5 保存图片
---
## 1. 需求
最近产品大大又又又搞事情，说什么想要在微信H5项目中做一个医生邀请提问的功能，可以将医生的二维码名片分享出去，之前图片由后端生成并且会缓存三天导致信息更新不及时；

我一听，这好说，不就是个保存图片的功能么，简单的看看需求：
 * 完善卡片信息，分享出去时候信息更加立体
 * 编辑个人资料入口
 * 保存图片入口
 * 可解决医生名片缓存时间问题
 * 长下面这样 ⬇

![image](http://ozwfe54j8.bkt.clouddn.com/note/wechat-sava-imageWX20171124-102804@2x.png)

分析下来就两点
 * html展示实时用户信息
 * 点击保存将当前页面保存成图片至本地，并且不包含功能按钮

## 2. 方案
因为之前已经听说过有个库能将HTML转为canvas，然后又听说canvas能转为图片，然后又听说图片能下载....(开发基本靠听说，这是废话)

那我的基本方案就是：
html -> canvas -> image -> a[download]

 1. html2canvas.js：可将htmldom转为canvas元素
 2. canvasAPI：toDataUrl()可将canvas转为base64格式
 3. 创建a[download]标签触发click事件实现下载

## 3. 开发
既然方案定下来那么说干就干，下面请开始我的踩坑表演，👏

### 3.1 [html2canvas.js](https://html2canvas.hertzen.com/documentation.html)
官方是这样介绍的：
>js将遍历加载页面的DOM节点，收集所有元素的信息，然后用这些信息来呈现页面。换句话说，实际上这个库并不是真的对页面进行截图，而是基于从DOM读取的元素及属性来一点点的绘制canvas。 因此，它只能正确地呈现它理解的元素和属性，这意味着有许多CSS属性不起作用。

```js
// v0.4.1
html2canvas(element, {
    onrendered: function(canvas) {
        // 现在你已经拿到了canvas DOM元素
    }
});

// v0.5.0
html2canvas(element, options).then(canvas => {
    // 现在你已经拿到了canvas DOM元素
});
```

#### 3.1.1 原理
该库的原理是将选中的dom及所有子节点全部跑一边，将每个节点用对应的方法挨个绘制到canvas中。

所以基本可以猜到整个工作流程应该是：

 1. 递归处理每个节点，记录这个节点应该怎么画。（比如div就画边框和背景，文字就画文字等等）
 2. 考虑节点的层级问题。比如z-index，float, position等样式的影响。
 3. 从低层级开始画到canvas上，逐渐向上画。层级高的覆盖层级低的。

#### 3.1.2 坑
目前官方提供的版本有很多，正式版本是`v0.4.1 - 7.9.2013`，最新版本是`v0.5.0-beta4`，那对于我们开发来说如果不是玩新特性什么的一般还是会选择正式版，结果第一个坑就掉进去爬了半天。。

##### 3.1.2.1 图片模糊
因为开发的时候是用chrome模拟器所以生成canvas后没有发现有模糊的地方，但是用pc代理手机请求开发资源时很明显的发现画面模糊的感觉非常明显

手机端截图，有很明显的锯齿感

![image](http://ozwfe54j8.bkt.clouddn.com/note/wechat-sava-imageWX20171124-102735@2x.png)

那么就想到可能是移动端像素密度计算的问题

> 设备像素比(简称dpr)定义了物理像素和设备独立像素的对应关系，它的值可以按如下的公式的得到：

`设备像素比 = 物理像素 / 设备独立像素 // 在某一方向上，x方向或者y方向`

知道了这个也没用，因为文档中根本没有给出能够配置像素比的地方。。

那么通过搜索后发现，官方文档其实还是`0.4.1`的，从`0.5.0`版本开始其实已经支持自定义canvas作为配置项传入了，它会根据我们传入的canvas为基础开始绘制。所以我们在调用html2canvas的时候，可以先创建好一个尺寸合适的canvas，再传进去。

那么话不多说，首先将库升级到`0.5.0`
```js
/**
 * 根据window.devicePixelRatio获取像素比
 */
function DPR() {
    if (window.devicePixelRatio && window.devicePixelRatio > 1) {
        return window.devicePixelRatio;
    }
    return 1;
}
/**
 *  将传入值转为整数
 */
function parseValue(value) {
    return parseInt(value, 10);
};
/**
 * 绘制canvas
 */
async function drawCanvas(selector) {
    // 获取想要转换的 DOM 节点
    const dom = document.querySelector(selector);
    const box = window.getComputedStyle(dom);
    // DOM 节点计算后宽高
    const width = parseValue(box.width);
    const height = parseValue(box.height);
    // 获取像素比
    const scaleBy = DPR();
    // 创建自定义 canvas 元素
    const canvas = document.createElement('canvas');

    // 设定 canvas 元素属性宽高为 DOM 节点宽高 * 像素比
    canvas.width = width * scaleBy;
    canvas.height = height * scaleBy;
    // 设定 canvas css宽高为 DOM 节点宽高
    canvas.style.width = `${width}px`;
    canvas.style.height = `${height}px`;
    // 获取画笔
    const context = canvas.getContext('2d');

    // 将所有绘制内容放大像素比倍
    context.scale(scaleBy, scaleBy);

    // 将自定义 canvas 作为配置项传入，开始绘制
    return await html2canvas(dom, {canvas});
}
```

手机端截图，和html展示效果一致，基本看不出来差别

![image](http://ozwfe54j8.bkt.clouddn.com/note/wechat-sava-imageWX20171124-102721@2x.png)

##### 3.1.2.2 <img>图片画出来怎么不见了

PC端截图

![image](http://ozwfe54j8.bkt.clouddn.com/note/wechat-sava-imageWX20171124-103407.png)

可能有多种原因，最主要的原因是图片跨域并且服务器未设置允许跨域？？[这里有解释](https://developer.mozilla.org/zh-CN/docs/Web/HTML/CORS_enabled_image)

看完发现我们好像需要做两件事：
 1. 给img元素设置`crossOrigin`属性，值为`anonymous`
 2. 图片服务端设置允许跨域

第一件事其实忽略，因为html2canvas支持配置`useCORS: true`；

但是第二件事有点难办，因为我们的图片一般都是上传到CDN上，而CDN为了更快的响应，会缓存图片的返回值，而缓存的值是不带跨域的头的。因为没有跨域的头，所以js请求会被拦截。但这是cdn不在我们手里的情况，如果在了那不就是后端改个配置的事情么，哈哈哈哈。

PC端： 完美。

微信环境下如果使用`0.5.0`也没有问题，但是使用`0.4.1`时绘制canvas的还是会导致图片丢失。。只能猜测是在html2canvas在预载图片和绘制图片时少了不可描述的东西。

因为一开始使用`0.4.1`所以这个坑我没有绕过去，强行解决：
```js
// 请求图片的事自己来做；将图片转为base64之后放回img的src中再进行绘制
/**
 * 图片转base64格式
 */
img2base64(url, crossOrigin) {
    return new Promise(resolve => {
        const img = new Image();

        img.onload = () => {
            const c = document.createElement('canvas');

            c.width = img.naturalWidth;
            c.height = img.naturalHeight;

            const cxt = c.getContext('2d');

            cxt.drawImage(img, 0, 0);
            // 得到图片的base64编码数据
            resolve(c.toDataURL('image/png'));
        };

        crossOrigin && img.setAttribute('crossOrigin', crossOrigin);
        img.src = url;
    });
}
```

**！注意：前提是服务端必须支持跨域，如果是无法改变服务端配置的图片最好提前砍掉，比如绘制微信头像**

##### 3.1.2.3 倒角
`border-radius` 只能是≤短边长度的一半，并且是具体数值，否则可能会出现奇妙的效果。

另外使用伪元素实现0.5px边框也可能会出现奇妙效果，建议直接使用border属性

`0.4.1`版本中需要做圆形图片只能置为背景图，img不支持绘制`border-radius`，`0.5`中无此限制

##### 3.1.2.4 虚线
使用`border-style: dashed/dotted`无效，还是大实线，切图在PC端有效，微信中需转为base64并且依赖环境，可能无效。

### 3.2 toDataUrl()
只要canvas中没有跨域图片可以随便转

**But** 在微信中就算没有也可能失败，在尝试使用切图渲染虚线时微信中还是会报`SecurityError, The operation is insecure.`错误，导致转base64失败

### 3.3 保存

理想
```js
/**
 * 在本地进行文件保存
 * @param  {String} data     要保存到本地的图片数据
 * @param  {String} filename 文件名
 */
saveFile(data, filename) {
    const save_link = document.createElementNS('http://www.w3.org/1999/xhtml', 'a');
    save_link.href = data;
    save_link.download = filename;

    const event = document.createEvent('MouseEvents');
    event.initMouseEvent('click', true, false, window, 0, 0, 0, 0, 0, false, false, false, false, 0, null);
    save_link.dispatchEvent(event);
}
```
现实

PC端： 完美。**微信**：不好意思，你说什么？我听不见？？！！

微信中根本没有任何反应。查看[微信sdk](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421141115)后发现：

 * `downloadImage`仅支持`uploadImage`接口上传的图片
 * `uploadImage`接口仅支持`chooseImage`接口相册选择的图片
 * `chooseImage`接口....
 * 妈蛋都在相册了还玩个毛线。
 * ....

## 4. 交付

最终实现的方案是：

* 用户进入该页面
* 获取当前用户所有信息，头像，二维码等
* 将所有图片转为base64
* 渲染`html`
* 绘制`canvas`
* 转为base64
* 替换`html`为`img`，`src`为base64
* 完成页面到图片的转换，微信中用户可长按页面调起actionSheet识别或保存图片
*
在回头看我们的需求
>  * html展示实时用户信息
>  * 点击保存将当前页面保存成图片至本地

其实只实现了第一点，第二点仅实现了一半，保存功能还是需要用户调起微信内置API来完成，而不是我们帮用户完成；微信他不给这个接口我也很绝望啊/(ㄒoㄒ)/~~

希望以上内容能够对大家以后的开发有所帮助，谢谢。

完。