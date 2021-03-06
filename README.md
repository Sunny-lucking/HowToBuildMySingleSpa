## 准备工作
1. 先创建一个项目文件夹：**howToBuildMySingleSpa**
2. 再创建`index.html` 文件
3. 创建`README.md` (非必要，这个文件是我用来写这篇文章的)

ok，截图看看现在的目录情况吧。


![](https://files.mdnice.com/user/3934/2ed4d194-d850-4cb4-8fc1-7c9fb94d4426.png)

## 使用single-spa
先在index.html中引入single-spa

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <script src="https://cdn.jsdelivr.net/npm/single-spa@5.9.3/lib/umd/single-spa.min.js"></script>
</body>
</html>
```
引入之后，就会在window中添加了个singleSpa对象。

接下来，我们注册两个微应用appA和appB。


```js
let appA = {
    bootstrap: [
        async () => {
            console.log("APP A bootstrap ----1");
        },
        async () => {
            console.log("APP A bootstrap ----2");
        }
    ],
    mount: async () => {
        console.log("APP A mount");
    },
    unmount: async () => {
        console.log("APP A unmount");
    }
}
let appB = {
    bootstrap: [
        async () => {
            console.log("APP B bootstrap ----1");
        },
        async () => {
            console.log("APP B bootstrap ----2");
        }
    ],
    mount: async () => {
        console.log("APP B mount");
    },
    unmount: async () => {
        console.log("APP B unmount");
    }
}
```

**微应用有一个特点，就是需要有bootstrap，mount，unmount三个生命周期函数。**

1. bootstrap是启动微应用的时候触发的。（只会触发一次）
2. mount是当微应用挂载到页面上的时候出发的。
3. unmount是卸载的时候触发。

**需要注意的是，每个周期函数，可以 是方法数组(如上面的bootstrap)，也可以是一个方法（如mount和unmount）。**

接下来，引入registerApplication方法来注册这两个应用和start方法来启动。

```js
let { registerApplication, start } = singleSpa;

···

registerApplication({
    name: "APPA",
    app: async () => { return appA },
    activeWhen: location => location.hash == "#/a",
    customProps: { author: "前端阳光" }
})
registerApplication(
    "APPB",
    async () => { return appB },
    location => location.hash == "#/b",
    { author: "前端阳光" }
)
start();
```
可以看到registerApplication接受四个参数，分别是：
1. `name`微应用的名称，用于防止注册重复的微应用，
2. `app`用于加载对应的微应用
3. `activeWhen`判断路径匹配的时候，渲染该微应用
4. `customProps`传递为微应用的参数。

可以以对象的形式传递参数（如上面注册APP A），也可以四个参数拆出来传递（如上面注册APP B）

然后执行start方法启动。

我们看看运行结果：

运行index.html
![](https://files.mdnice.com/user/3934/621a8c22-5b7e-4973-803d-5501797eb9f4.png)

然后点击App A


![](https://files.mdnice.com/user/3934/7cb3d5f0-048f-47b5-a1eb-9afee7c37009.png)

再点击App B

![](https://files.mdnice.com/user/3934/db0ab609-3b4b-4ad0-ad3b-812227299250.png)

再点击回APP A

![](https://files.mdnice.com/user/3934/039fb2e7-80f6-4e69-9b41-aa1f29ed4d4b.png)

可以看到 当点击App A 的时候，url的hash值 跟appA应用的路径匹配对上了，然后就启动APPA，再挂载AppA

当点击App B 的时候，url的hash值 跟appB应用的路径匹配对上了，然后就卸载APPA，启动APPB，再挂载AppB。

当再点击回APP A ，appA的bootstrap就不会再被触发了。

ok，对于single-spa的基本使用已经介绍完毕了，接下来，就自己来实现一个single-spa吧！

>先把目前的代码上传到github：https://github.com/Sunny-lucking/HowToBuildMySingleSpa/tree/b044a6caa1cf6e14ea37847403e0d9e2b893b5b6

## 创建自己的single-spa

如图所示：
1. 创建single-spa文件夹作为我们自己实现的single-spa。
2. start.js存放的是 start方法的实现
3. applications存放的是跟应用相关的文件，例如apps.js存放的是registerApplication方法的实现
4. single-spa.js是入口文件，引入start和registerApplication并导出

![](https://files.mdnice.com/user/3934/607ad2ee-136c-44a2-b504-266aa7e5e175.png)

**start.js**


```js
export function start() {
    console.log("start");
}
```

**apps.js**

```js
export function registerApplication() {
    console.log('register');
}
```
**single-spa.js**

```js
export { registerApplication } from "./applications/apps.js"
export { start } from "./start.js"
```
>先把目前的代码上传到github：https://github.com/Sunny-lucking/HowToBuildMySingleSpa/tree/2505bb604d0ed66973f6f81da3b8c226bdc76d11

## 实现应用状态

我们先看一下一个微应用一生中会经历的状态。

为了更好的管理app，特地给app增加了状态，每个app共存在11个状态，其中每个状态的流转图如下：
![](https://files.mdnice.com/user/3934/0d79169b-eb1b-4a89-ace4-82d9de82a345.png)

每个状态的介绍如下
![](https://files.mdnice.com/user/3934/474914e5-e94d-46aa-bb6e-db5ad8df8f7e.png)

因此，我们专门在application文件夹下创建一个app.helper.js文件来定义这些状态

我们实现最主要的八个状态就好了

**app.helper.js**

```js
export const NOT_LOADED = "NOT_LOADED"; // 没有加载过
export const LOADING_SOURCE_CODE = "LOADING_SOURCE_CODE"; // 加载原代码
export const NOT_BOOTSTRAPPED = "NOT_BOOTSTRAPPED"; // 没有启动
export const BOOTSTRAPPING = "BOOTSTRAPPING"; // 启动中
export const NOT_MOUNTED = "NOT_MOUNTED"; // 没有挂载
export const MOUNTING = "MOUNTING"; // 挂载中
export const MOUNTED = "MOUNTED"; // 挂载完毕
export const UNMOUNTING = "UNMOUNTING"; // 卸载中
```
接下来，在这个文件夹中实现两个方法，

**isActive**判断判断该应用是否已被挂载（判断该应用的状态是否为MOUNTED即可）；

**shouldBeActive**判断该应用是否应该被加载（判断当前的路径是否匹配即可）


```js
// 判断判断该应用是否已被挂载
export function isActive(app) {
    return app.status === MOUNTED;
}

// 判断该应用是否应该被加载
export function shouldBeActive(app) {
    return app.activeWhen(window.location)
}
```

相信会有同学有疑问，我们定义app的时候，没有staus这个属性啊，没错这个是registerApplication方法里面给每一个应用添加上的。

接下来我们就实现registerApplication方法。

>先把目前的代码上传到github：https://github.com/Sunny-lucking/HowToBuildMySingleSpa/tree/e191043b6d99814e95843941c8ac86bcd21276eb

## 实现registerApplication方法


```js
import { NOT_LOADED } from "./app.helper.js";

/**
 * 
 * @param {*} name 应用昵称
 * @param {*} loadApp 加载应用
 * @param {*} activeWhen 匹配路径
 * @param {*} customProps 参数
 */
let app = [] // 将所有注册的应用都放在一起
export function registerApplication(name, loadApp, activeWhen, customProps) {
    let registeration;
    if(typeof name === "object"){ // 参数是对象的形式
        registeration = appName
    }else {
        registeration = {
            name,
            loadApp,
            activeWhen,
            customProps
        }
    }
    // 添加状态
    registeration.status = NOT_LOADED
    app.push(registeration);
}
```
如代码所示，用一个apps来将这些注册的应用到放在了一起。

其中需要判断是以那种方式传参的，我们在前面介绍过，registerApplication支持两种传参方式，所以都要兼容。

注册完之后，就需要加载应该被加载的应用了。

加载应用的逻辑放在一个reroute方法里。


```js
let app = [] // 将所有注册的应用都放在一起
export function registerApplication(name, loadApp, activeWhen, customProps) {
    let registeration;
    if(typeof name === "object"){ // 参数是对象的形式
        registeration = appName
    }else {
        registeration = {
            name,
            loadApp,
            activeWhen,
            customProps
        }
    }
    // 添加状态
    registeration.status = NOT_LOADED
    app.push(registeration);

    reroute();// 重写路径，后续切换路由要再做这些事情
}
```
之所以叫reroute，是因为每次路由更换的时候，都要执行reroute方法。

由于reroute方法在registerApplication和start中都要用到，所以我们单纯创建一个文件来实现这个方法。

**/navigation/reroute.js**


![](https://files.mdnice.com/user/3934/8a77d6f9-e4d3-4f88-ae39-57d69f1660a1.png)

>先把目前的代码上传到github：