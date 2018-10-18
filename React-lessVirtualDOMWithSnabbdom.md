# [译] 使用 sanbbdom 实现无 react 的虚拟 DOM：函数无所不能！

###### 原文链接：[https://medium.com/@yelouafi/react-less-virtual-dom-with-snabbdom-functions-everywhere-53b672cb2fe3](https://medium.com/@yelouafi/react-less-virtual-dom-with-snabbdom-functions-everywhere-53b672cb2fe3)

React 对于 JavaScript 社区做了真正意义上的补充。 虽然人们可以在 JavaScript 里写存在一定争议性的 JSX（类似 HTML 的语法），但是这和 Virtual DOM 的概念不一样。

笼统而言，Virtual DOM 是物理 DOM 在某个时刻的状态的一种简化的内存代表。其思想是：不是直接用命令语句去更新真正的 DOM，而是每次在你的 UI 状态变化的时候，构建一个虚拟的 DOM，底层的库会相应更新物理的 DOM。

需要注意的是，更新操作并不替换整个 DOM 树（比如用 HTML 字符串设置 innerHTML），而只替换 DOM 实际更改的部分（修改 Node 属性，添加子元素…）。即，通过对比旧的和新的虚拟 DOM 版本，来替换真实的 DOM 以反映新版本的变化。

Virtual DOM 因其快速的性能而受到人们的重视。但也有一个同样重要的概念--属性。该概念使得可以将 UI 表示为其状态的函数。这使得探索 Web 应用程序编写的新方法成为可能。

在这篇文章中，我们将研究 Virtual DOM 的概念在 Web 应用程序中的适用性。我们将从简单的示例开始，然后编写一个基于 Virtual DOM 框架的应用程序（提示：如果你喜欢纯函数，就不会失望）。

因此，我们将选择一个独立和简单的 JavaScript（没有 JSX）Virtual DOM 库，因为我们需要研究最小的基本谜题。

周围有一些好的独立的 Virtual DOM 库，在我的文章里，我将使用 Simon Friis Vindum 写的一个很小的库 snabbdom [(paldepind/snabbdom)](https://github.com/snabbdom/snabbdom),但是这个例子可以通过其他任何类似的库实现，例如 Matt Esch 的[virtual-dom](https://github.com/Matt-Esch/virtual-dom)

### snabbdom 快速入门

snabbdom 是以 CommonJs modules 的方式提供的，所以我们需要一个客户端模块打包工具，例如 browserify 或者 webpack。

首先让我们来看看怎样引导一个基本的应用程序

    import snabbdom from 'snabbdom';

    const patch = snabbdom.init([                   // Init patch function with choosen modules
        require('snabbdom/modules/class'),          // makes it easy to toggle classes
        require('snabbdom/modules/props'),          // for setting properties on DOM elements
        require('snabbdom/modules/style'),          // handles styling on elements with support for animations
        require('snabbdom/modules/eventlisteners'), // attaches event listeners
    ]);

在上面我们通过一组扩展项初始化了 snabbdom 的核心模块。在 snabbdom 中，像在 DOM 元素上切换 class、style 甚至设置属性之类的功能都委托给单独的模块。对我们而言，我们将只使用默认模块。

核心模块只暴露了一个“patch”函数，由 init 方法返回。我们将使用它来创建以及更新初始 DOM。

这是一个通过 snabbdom 编写的基本的 Hello 例子

    import h from 'snabbdom/h';

    var vnode = h('div', {style: {fontWeight: 'bold'}}, 'Hello world');
    patch(document.getElementById('placeholder'), vnode);

'h‘是用来创建虚拟节点的辅助函数，我们将在本文的剩余部分讨论它的用法，现在只需要记住它的三个参数：

1. 一个选择器例如‘div#id.class’
2. （可选）数据对象，它指定虚拟节点的属性（类，样式，事件......），稍后会看到。
3. （可选）一个字符串或包含虚拟子节点的数组

首先，patch 期望传入一个（例如 id 为 placeholder）DOM 元素和一个初始的虚拟节点。然后创建初始的 DOM 树来映射虚拟的 DOM 树。在随后的调用中，我们为它传入之前和新的虚拟节点。然后，它对 2 个虚拟节点进行对比，再在 DOM 中进行必要的修改，将当前 DOM 新的状态映射成新的虚拟表达。

为了快速地启动项目，我创建了一个 GitHub 仓库，包含所有基本配置并且可以马上启动。因此，第一步是在[(YelouaFi/SababdOn)](https://github.com/yelouafi/snabbdom-starter)克隆这个仓库。之后，通过 npm install 安装所有的依赖。该依赖包使用 Browserify 进行模块打包，Watchify 进行文件更改的自动构建，Babel 允许我们编写不错的 ES6 代码并将其转换为 ES5 兼容的代码。

安装好后输入

    npm run watch

这个命令将启动 Watchify 模块，它将在 app 文件夹里创建一个 build.js 文件，并且它可以监听我们在 JavaScript 文件里的更改进行自动构建（你也可以输入“npm run build”进行手动构建）

若要运行该应用程序，请在浏览器中打开“app/index .html”文件。通常，你应该在屏幕上看到一个“Hello World”消息。

我们在本文中看到的所有例子都可以在这个仓库找到，每一个例子有一个与之对应的分支，我们在文中会相应的链接到对应的分支，这个仓库的 README 文件也包含了所有分支的链接。

#### 动态视图

> 这个例子的来源在[(dynamic-view)](https://github.com/yelouafi/snabbdom-starter/tree/dynamic-view)分支

为了突出 Virtual DOM 的动态特点，我们将构建一个非常简单的时钟。更改'app / js / main.js'如下

    function view(currentDate) {
        return h('div', 'Current date ' + currentDate);
    }

    var oldVnode = document.getElementById('placeholder');

    setInterval(() => {
        const newVnode = view(new Date());
        oldVnode = patch(oldVnode, newVnode);
    }, 1000);

通过一个单独的函数“view”来构造虚拟节点，并且将当前状态（当前日期）作为输入。

该示例说明了基于虚拟 DOM 的应用程序中的典型模式。在底层，应用程序行为包括在关注的不同时刻（计时器、用户事件、服务器事件…）连续构建新的虚拟树，然后用与最新虚拟节点相比的差异来更改实际的 DOM。在我们的例子中，我们每秒构造一个新的虚拟树并用它更新 DOM。

#### 事件反应

> 这个例子的来源在[(event-reactivity)](https://github.com/yelouafi/snabbdom-starter/tree/event-reactivity)分支

下一个示例通过一个基本问候语应用程序介绍事件处理

    function view(name) {
        return h('div', [
            h('input', {
                props: { type: 'text', placeholder: 'Type  your name' },
            on   : { input: update }
            }),
            h('hr'),
            h('div', 'Hello ' + name)
        ]);
    }

    var oldVnode = document.getElementById('placeholder');

    function update(event) {
        const newVnode = view(event.target.value);
        oldVnode = patch(oldVnode, newVnode);
    }


    oldVnode = patch(oldVnode, view(''));

在 snabbdom 中，我们通过'props'对象设置元素的属性，这些对象由'props'模块处理。类似地，我们通过'on'对象设置事件监听器，该对象由'eventlisteners'模块处理。

在上面的示例中，“update”函数发挥与前一示例中的“setInterval”相同的作用：它从事件中获取数据来构造新的虚拟树，然后调用 patch 用新创建的树更新 DOM。

### 构建复杂的应用程序
