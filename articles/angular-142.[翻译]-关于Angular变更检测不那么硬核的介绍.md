# [翻译] 关于 Angular 变更检测不那么硬核的介绍

> 原文链接：[A gentle introduction into change detection in Angular](https://blog.angularindepth.com/a-gentle-introduction-into-change-detection-in-angular-33f9ffff6f10)
>
> 原文作者：[Max Koretskyi，aka Wizard](https://blog.angularindepth.com/@maxim.koretskyi?source=post_header_lockup)
>
> *原技术博文由 [`Max Koretskyi`](https://twitter.com/maxim_koretskyi) 撰写发布，他目前于 [ag-Grid](https://angular-grid.ag-grid.com/?utm_source=medium&utm_medium=blog&utm_campaign=angularcustom) 担任开发者职位。*
>
> 译者按：开发大使负责确保其所在的公司认真听取社区的声音并向社区传达他们的行动及目标，其作为社区和公司之间的纽带存在。
> 
> 译者：[尊重](https://www.zhihu.com/people/yiji-yiben-ming/activities)；校对者：[baishusama](https://github.com/baishusama)
>
> 本文是一篇是关于「变更检测机制」，「zones」和「`ExpressionChangedAfterItHasBeenCheckedError` 错误」的高层次概述。

<p align="center"> 
    <img src="../assets/angular-142/1.jpeg">
</p>

> 如果你更希望通过视频的方式了解这篇文章，可以查看[原作者在 *AngularConnect* 上所做的演讲](https://www.youtube.com/watch?v=DsBy9O0c6eo)。

现代 Web 应用都是交互式的，web 应用的状态随时都可能发生变化，而产生变化的原因可能是一个按钮的点击或从服务器返回的一个请求。代码需要检测到应用的状态发生变化并及时将变化反射到用户 UI 界面上，而[这就是变更检测机制的主要工作](https://medium.freecodecamp.org/what-every-front-end-developer-should-know-about-change-detection-in-angular-and-react-508f83f58c6a)。

在过去的一年中，我已经写了一系列有关 Angular 变更检测的 [InDepth 文章](https://blog.angularindepth.com/these-5-articles-will-make-you-an-angular-change-detection-expert-ed530d28930)。这些文章对变更检测给出了详细的解释并且覆盖了很多内部的相关细节，但是这也意味着你可能需要花费不少的时间去读完，读透全部的内容。对于那些没有那么多时间但是仍然对 Angular 变更检测好奇的读者，本文将会以轻量级的方式介绍变更检测机制。本文将会从高层次给出一个关于变更检测的组成部分和机制的概述：代表组件的内部数据结构，绑定在其中扮演的角色以及该流程中的操作。我也会涉及 `zones` 的概念并向你准确地展示其如何在 Angular 中启用自动变更检测。

当 Angular 应用出错时，了解变更检测的内部机理会帮助你更加有效地 debug （像是 [`ExpressionChangedAfterItHasBeenCheckedError`](https://blog.angularindepth.com/everything-you-need-to-know-about-the-expressionchangedafterithasbeencheckederror-error-e3fd9ce7dbb4) 这样的）错误，帮你避免[一些常见的困惑](https://blog.angularindepth.com/if-you-think-ngdocheck-means-your-component-is-being-checked-read-this-article-36ce63a3f3e5)。在本文中，我将演示一些导致错误的设置，并使用它们来解释变更检测及其内部的相关内容。

## 初次接触

让我们从一个简单的 Angular 组件开始，它呈现了在应用中发生变更检测时的时刻，其时间戳具有毫秒的精度。点击按钮以触发变更检测：

<p align="center"> 
    <img src="../assets/angular-142/2.gif">
</p>

> [Demo 地址](https://stackblitz.com/edit/angular-hqbenm?file=src/app/app.component.ts)

以下为实现：

```typescript
@Component({
  selector："my-app"，
  template：`
    <h3>
      Change detection is triggered at:
      <span [textContent]="time | date：'hh:mm:ss:SSS'"></span>
    </h3>
    <button (click)="(0)">Trigger Change Detection</button>
  `
})
export class AppComponent {
  get time() {
    return Date.now();
  }
}
```

正如你所见的那样这是一个很简单的实践，有一个名为 `time` 的 `getter` 函数返回当前时间戳，它绑定到了 HTML 中的 `span` 元素上。

> Angular 并不允许空的表达式，所以这里使用 `0` 作为点击事件的回调。

你可以在[这里](https://stackblitz.com/edit/angular-hqbenm?file=src/app/app.component.ts)进行你自己的尝试。当 Angular 执行变更检测时，它获取了 `time` 属性的值将它传递给 `date` 管道并使用其结果更新 DOM 。目前为止一切都正常运作。然而当我检查控制台的时候看到了`ExpressionChangedAfterItHasBeenCheckedError` 这个错误：

<p align="center"> 
    <img src="../assets/angular-142/3.png">
</p>

这着实让人惊讶，通常而言这个错误会伴随更加复杂的场景出现。那么我们是如何通过如此简单的功能产生了这个错误呢？别担心，我们好好调查一下。

让我们从错误信息开始：

> Expression has changed after it was checked. Previous value：“textContent：1542375826274”. Current value：“textContent：1542375826275”.

这条错误信息告诉我们用于 `textContent` 绑定的表达式生成的值是不同的。是的，毫秒数的确不同。所以 Angular 评估了表达式 `time | date:’hh:mm:ss:SSS` 两次并比较了两次的结果。Angular 检测到了不同，并因此触发了错误。

### 但为什么 Angular 会进行这种比较呢？

#### 或这种比较究竟是什么时候做的呢？

这些问题引起了我的好奇心，并最终导致我钻入变更检测的内部世界。为了找到这些问题的答案，我必须开始调试，于是我度过了长达几个月不停调试的时光 😅……让我们从第二个问题（这个错误何时被抛出）开始，但在那之前，我需要与你分享一些我的发现，这些发现将有助于理解上面那些我们刚才观察到的表现。

## 组件视图和绑定

在 Angular 中有两个关于变更检测的主要构建块（building blocks）：

- 组件视图（a component view）
- 相关绑定（the associated bindings）

> 译者注：因为[屏幕上的一小片区域也被称为“视图”](https://angular.cn/guide/architecture-components#introduction-to-components)，而本文中的 view 基本上都指的是一个 JS 对象。为了不产生歧义，本译文对 view 全文不翻译，只对 component view 翻译为组件视图。

Angular 中的每个组件都包含一个带有 HTML 元素的模版。当 Angular 创建 DOM 节点以在屏幕上呈现模板的内容时，它需要一个位置来存储这些 DOM 节点的引用。为此，内部有一个称为「View」的数据结构。它也用于存储组件实例的引用以及绑定表达式（binding expressions）的先前的值（previous values）。组件和 view 之间存在一对一的关系。下图示意了 view：

<p align="center"> 
    <img src="../assets/angular-142/4.png">
</p>

当编译器分析模板时，它会识别在变更检测期间可能需要更新的 DOM 元素的属性，对于每个这样的属性，编译器都会创建一个「绑定（binding）」。绑定定义了要更新的属性名称和 Angular 用于获取新值的表达式。

在我们的示例中，属性 `time` 在属性 `textContent` 的表达式中被使用，因此 Angular 创建了一个绑定并将其与 `span` 元素相关联：

<p align="center"> 
    <img src="../assets/angular-142/5.png">
</p>

> 在实际的实现中绑定并不是具有所有必需信息的单个对象， `viewDefinition` 定义了模板元素的实际绑定和要更新的属性，而用于绑定的表达式放在 `updateRenderer` 函数中。

## 检查组件视图

如你所知，Angular 中的每个组件都会执行变更检测，而现在我们知道组件在内部被表示为 view，我们可以说每个 view 都执行了变更检测。

当 Angular 检查 view 时，它只是运行编译器为 view 生成的所有绑定。它计算表达式并将其结果与存储在 view 上的 `oldValues` 数组中的值进行比较。这就是脏检查其名的来源。如果它检测到差异，则更新与绑定相关的 DOM 属性，并将新的值放入 view 的 `oldValues` 数组中。这就是全部啦～现在，你就拥有了一个更新过的 UI。一旦 Angular 完成了对于当前组件的检查，它就会为子组件重复完全相同的变更检测步骤。

在之前的应用中，只有一个到 App 组件中的 `span` 元素上的属性 `textContent` 的绑定。因此在变更检测期间，Angular 会读取组件属性 `time` 的值，应用 `date` 管道，并将其与存储在 view 上的先前的值进行比较。如果它检测到差异，Angular 将更新 `span` 的 `textContent` 属性和 `oldValues` 数组的内容。

### 但是错误从何而来？

在开发模式下，每个变更检测周期之后，Angular 会**同步**运行另一个检查以确保表达式生成与上一次变更检测运行结果相同的值。这个检查不是原始变更检测周期的一部分。它在完成对整个组件树的检查**之后**运行，并执行完全相同的步骤。只是这次当 Angular 检测到差异时它不会更新 DOM，而是抛出一个 `ExpressionChangedAfterItHasBeenCheckedError` 错误。

<p align="center"> 
    <img src="../assets/angular-142/6.png">
</p>

### 原因

现在我们知道了错误抛出的时间， **但是为什么 Angular 需要这次检测呢？** 这样，我们想象一下，组件的某些属性在变更检测运行的期间发生了更新。结果，表达式产生了与渲染到用户界面的值不一样的新数据。那么此时 Angular 应该做什么？它当然可以运行另一个变更检测周期来同步应用状态到用户界面。但是如果在此过程中某些属性再次更新了呢？发现了吗，Angular 实际上可能会陷入一个无限循环的变更检测运行过程。而事实上，[这种糟糕的情况在 AngularJS 中时常发生](https://docs.angularjs.org/error/$rootScope/infdig)。

为了避免这种情况，Angular 强加了所谓的[单向数据流](https://blog.angularindepth.com/do-you-really-know-what-unidirectional-data-flow-means-in-angular-a6f55cefdc63)。在变更检测之后运行的同步检查、先前生成的 `ExpressionChangedAfterItHasBeenCheckedError` 错误，都是该强制机制的体现。一旦 Angular 处理了当前组件的绑定，你就不能再更新在绑定表达式中使用的组件的属性了。

## 修复错误

为了防止错误，我们需要确保在变更检测运行期间表达式返回的值和之后的检查是相同的。在我们的例子中，我们可以通过将赋值的部分移出 `time` 的 `getter` 函数来实现：

```typescript
export class AppComponent {
    _time;
    get time() {  return this._time; }

    constructor() {
        this._time = Date.now();
    }
}
```

然而若是通过此种方式实现，`time` 的 `getter` 函数返回的的值将始终相同，但是我们仍旧希望更新该值。我们先前已了解到，产生错误的检查会在变更检测周期后立刻**同步**执行。因此如果我们**异步**地更新该值，将能够避免错误。因此，为了每毫秒更新一次 `_time` 值，我们可以使用 `setInterval` 函数并设置延迟为 1 毫秒，如下所示：

```typescript
export class AppComponent {
    _time;
    get time() {  return this._time; }

    constructor() {
        this._time = Date.now();
        setInterval(() => {
            this._time = Date.now();
        }，1);
    }
}
```

这样的实现方式解决了我们的初始问题，但是很不幸的是，这又带来了一个新的问题。所有计时事件（如 `setInterval`，`setTimeout`）都会在 Angular 中触发变更检测。这意味着通过这种实现，我们最终将进入一个无限循环的变更检测周期之中。 **为避免这种情况，我们需要一种方法既执行 `setInterval` 又不触发变更检测机制。** 幸运的是有一种方法可以做到这一点。要了解如何做到，我们首先需要了解为什么 `setInterval` 触发了 Angular 中的变更检测。

### 利用 zones 实现的自动化变更检测

与 React 相反，Angular 中的变更检测机制可以随着浏览器中的任何异步事件的发生而全自动触发。这种全自动之所以可能，是因为引入了包含 zone 概念的 [`zone.js`](https://blog.angularindepth.com/i-reverse-engineered-zones-zone-js-and-here-is-what-ive-found-1f48dc87659b) 库。与流行的看法相反，zone 本身不是 Angular 变更检测机制的一部分。事实上，[Angular 就算没有 zone 也可以运行](https://blog.angularindepth.com/do-you-still-think-that-ngzone-zone-js-is-required-for-change-detection-in-angular-16f7a575afef)。`zone.js` 这个库只是提供了一种拦截异步事件的方法（如`setInterval`，`setTimeout`）并通知 Angular 它们的发生。Angular 在这些通知的基础上运行变更检测。

有趣的是，你可以在一个网页上拥有许多不同的 zone，而其中一个将是 `NgZone`。`NgZone` 是在 Angular 项目引导（bootstrap）时创建的，Angular 应用就是在这个 zone 里运行的。而且，**Angular 仅仅获取有关这个 zone 内发生的事件的通知。**

<p align="center"> 
    <img src="../assets/angular-142/7.png">
</p>

但是，`zone.js` 也提供了一个 API 允许在 Angular 的 zone（NgZone）外的其它 zone 执行代码，而 Angular 不会收到有关这些其他 zone 中发生的异步事件的通知。而没有通知意味着不会触发变更检测。执行此操作的方法名称为 `runOutsideAngular`，它由 `NgZone` 服务实现。

下述是注入 `NgZone` 服务并在 Angular zone 之外运行 `setInterval` 的实现：

```typescript
export class AppComponent {
    _time;
    get time() {
        return this._time;
    }

    constructor(zone：NgZone) {
        this._time = Date.now();

        zone.runOutsideAngular(() => {
            setInterval(() => {
                this._time = Date.now()
            }，1);
        });
    }
}
```

现在，我们仍立即更新时间，**但我们是在 Angular zone 之外异步地这么做的。这样的方式保证了在变更检测和后续的同步检查中，`time` 属性的 `getter` 函数返回相同的值。并且当 Angular 在下一个变更检测周期中读取 `time` 属性的值时，该值将会更新并且更新将反应到用户界面上。**

> 使用 `NgZone` 在 Angular 的 zone 之外运行一些代码以避免触发变更检测是一种常见的优化技术。

## 调试

您可能想知道是否有任何方法可以查看 Angular 中的 view 和绑定，事实上是有的。`@angular/core` 模块中有一个名为 [`checkAndUpdateView`](https://github.com/maximusk/angular/blob/24cf8b326918e0694dfcdcf48635b5325060827a/packages/core/src/view/view.ts#L350) 的函数，它在组件树中的每个 view(component) 上运行，并对每个 view 执行检查。当我遇到任何与变更检测有关的问题时，这个方法总是我开始调试的起点。

请自己试着去调试看看吧。前往这个 [demo 地址](https://angular-eobrrh.stackblitz.io/)并打开控制台。在代码中找到该函数并设置一个断点，单击按钮以触发变更检测，检查 `view` 变量。以下是我在这么做时的记录：

<p align="center"> 
    <img src="../assets/angular-142/8.gif">
</p>

第一个 `view` 将是主 view（host view）。（译者注：主 view 对于其它 view 而言，）就像 Angular 为了托管我们的 app component 而创建的根组件（root component）。我们需要继续执行才能进入其子 view，而这个子 view 就是为 `AppComponent` 组件所创建的 view。探索一番。`component` 属性包含对 `App` 组件实例的引用，`nodes` 属性包含为 `App` 组件模板内的元素而创建的 DOM 节点的引用，而 `oldValues` 数组存储绑定表达式的结果。

## 操作执行的顺序

我们之前聊到，由于单向数据流的限制，你无法修改一个「在变更检测期间、已经被检查过的」组件的属性。大多数情况下，这种更新会通过「共享的服务或同步的事件广播」在 Angular 为子组件运行变更检测时发生。但是，你也可以通过直接将父组件注入子组件中的方式，于某个生命周期钩子中更新父组件的状态。如下代码所示：

```typescript
@Component({
    selector：'my-app'，
    template：`
        <div [textContent]="text"></div>
        <child-comp></child-comp>
    `
})
export class AppComponent {
    text = 'Original text in parent component';
}

@Component({
    selector：'child-comp'，
    template：`<span>I am child component</span>`
})
export class ChildComponent {
    constructor(private parent：AppComponent) {}

    ngAfterViewChecked() {
        this.parent.text = 'Updated text in parent component';
    }
}
```

你可以在这个 [demo 地址](https://stackblitz.com/edit/angular-zntusy)中实际演练上面的功能。简单来说，我们定义了两个组件间的层次结构，父组件声明了一个用于绑定的 `text` 属性，子组件在构造函数中注入了父组件并于 `ngAfterViewChecked` 生命周期钩子中更新了 `text` 属性的值。你猜猜如果我们此时打开控制台会看到什么？😃

是的，我们会看到熟悉的 `ExpressionChangedAfterItWasChecked` 错误，这是因为当 Angular 为子组件调用 `ngAfterViewChecked` 生命周期钩子时，Angular 已经完成了对于父组件 `App` 所拥有的绑定的检查（译者注：这里指变更检测周期内的检查，而不是之后的同步检查），而我们在检查结束后更新了绑定中使用的父组件的 `text` 属性的值。

但是现在有个有趣的问题，如果我现在更换生命周期钩子的话会发生什么？例如，对于 `ngOnInit` 生命周期钩子，你认为我们还会看到错误吗？

```typescript
export class ChildComponent {
    constructor(private parent：AppComponent) {}

    ngOnInit() {
        this.parent.text = 'Updated text in parent component';
    }
}
```

Wow，这次没有错误了，你可以通过这个 [demo](https://stackblitz.com/edit/angular-ti5abu) 查看相关的实现。实际上，我们可以将代码放在任何其他钩子中（`AfterViewInit` 除外），我们都将不会在控制台中看到错误。为什么 `ngAfterViewChecked` 钩子这么特别？

要理解这种状况，我们需要知道 Angular 在变更检测期间执行的操作及其顺序。而我们已经知道可以通过 `checkAndUpdateView` 方法来满足我们的需要。以下是可以在其函数体中找到的一部分代码：

```typescript
function checkAndUpdateView(view，...) {
    ...       
    // update input bindings on child views (components) & directives，
    // call NgOnInit，NgDoCheck and ngOnChanges hooks if needed
    Services.updateDirectives(view，CheckType.CheckAndUpdate);
    
    // DOM updates，perform rendering for the current view (component)
    Services.updateRenderer(view，CheckType.CheckAndUpdate);
    
    // run change detection on child views (components)
    execComponentViewsAction(view，ViewAction.CheckAndUpdate);
    
    // call AfterViewChecked and AfterViewInit hooks
    callLifecycleHooksChildrenFirst(…，NodeFlags.AfterViewChecked…);
    ...
}
```

正如你所见，Angular 在变更检测期间也触发了生命周期钩子。**有趣的是，某些生命周期钩子「在 Angular 处理绑定时、在渲染部分之前」被调用，另外一些钩子则会在之后被调用**。下面的图表演示了当 Angular 运行父组件的变更检测时会发生什么：

<p align="center"> 
    <img src="../assets/angular-142/9.png">
</p>

让我们一步步地去理解它。首先，它对**子组件**更新了输入绑定（input bindings），然后它再次对**子组件**调用了其 `OnInit`，`DoCheck` 和 `OnChanges` 生命周期钩子。这一过程是有道理的，因为它只是更新了输入绑定以及 Angular 需要通知子组件其输入绑定的初始化已经完成。**之后 Angular 对当前组件进行了渲染**。在渲染完成后，将会对子组件执行变更检测。这一过程意味着它基本上会在子 view 上重复上述操作。最终，它调用**子组件**上的 `AfterViewChecked` 和 `AfterViewInit` 生命周期钩子，让它知道自身已被检查。

我们能注意到一个要点：Angular 是在处理父组件的绑定**之后，才去调用子组件的 `AfterViewChecked`** 生命周期钩子的。而在另一方面，**`OnInit` 生命周期钩子则是在处理绑定之前**被调用的。因此，即使在 `OnInit` 钩子中修改 `text` 的属性值，在后续的同步检查中它的值（译者注：与 `oldValues` 数组内的属性相比较）仍然是相同的。这解释了看似奇怪的使用 `ngOnInit` 钩子避免了 error 的状况。问题解决了，神清气爽 🤓。

## 总结

最后，让我们总结一下我们都学到了什么。Angular 所有组件在内部都以称为 view 的数据结构中表示。 Angular 的解析器解析模板并创建绑定。每个绑定定义了一个需要更新的 DOM 元素的属性以及用于获取其值的表达式。在变更检测期间用于比较的先前的值（previous values）是存储在 view 的 `oldValues` 属性中的。在变更检测期间，Angular 执行了绑定、计算表达式、将它们与先前的值进行比较并在必要时更新 DOM。在每个变更检测周期之后，Angular 会（译者注：在开发模式下额外）运行一个检查以确保组件状态与用户界面内容同步。此检查是同步执行的，可能会抛出 `ExpressionChangedAfterItWasChecked` 错误。

## 来自原作者的寄语

我希望这些知识能唤醒你的好奇心，我想通过这种方式激励你了解有关 Web 平台，架构和编程的更多信息。我由衷希望你能成为一名非凡的工程师。