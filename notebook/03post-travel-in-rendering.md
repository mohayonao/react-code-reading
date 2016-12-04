# 第3回の復習

オンラインでは全然追いつけなかったので、改めてステップ実行しつつ調べる。

```html
<!DOCTYPE html>
<html>
  <head><title>travel in rendering</title></head>
  <body><div id="app"></div></body>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.4.0/react.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.4.0/react-dom.js"></script>
  <script>
    window.addEventListener("DOMContentLoaded", () => {
      const app = document.getElementById("app");
      const span = React.createElement("span", null, "hello");
      // {"type":"span","key":null,"ref":null,"props":{"children":"hello"},"_owner":null,"_store":{}}

      debugger;
      ReactDOM.render(span, app); // <- ここ！！
    });
  </script>
</html>
```

こういうDOMが生成される。  
ちなみに `React.createElement` は何らかのインスタンスではなくて、プレーンオブジェクトを返す。

```html
<div id="app">
  <span>hello</span>
</div>
```

---

## [ReactDOM.js](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/ReactDOM.js)

`ReactDOM.render` は `ReactMount.render` の別名。 [→](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/ReactDOM.js#L33)

---

## [ReactMount.js](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js)

### [ReactMount.render](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L552)

目的: `nextElement` を DOM化して `div#app` に追加する。

```
ReactMount.render(
  nextElement, container, callback
) |            |          |
  object[span] div#app    undefined
```

`ReactMount.render` は `ReactMount._renderSubtreeIntoContainer` を呼び出すだけ。 [→](
https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L552)

```js
return ReactMount._renderSubtreeIntoContainer(null, nextElement, container, callback);
```

### [ReactMount._renderSubtreeIntoContainer](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L434)

目的: DOMを部分的に更新する。

```
ReactMount._renderSubtreeIntoContainer(
  parentComponent, nextElement, container, callback
) |                |            |          |
  null             object[span] div#app    undefined
```

初回のレンダリングなので `parentComponent` が `null` である。

```js
var nextWrappedElement = React.createElement(
  TopLevelWrapper,
  { child: nextElement }
);
```

`nextElement` を `TopLevelWrapper` で囲った `nextWrappedElement` を生成。 [→](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L464)  

[TopLevelWrapper](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L268) は `render()` 時に `nextElement` を返すだけの `React.Component`。これは `nextElement.type` が `React.Component` の場合に定義されているメソッドの呼び出しを遅らせるために必要とのこと。

> Temporary (?) hack so that we can store all top-level pending updates on  
> composites instead of having to worry about different types of components  
> here.

```js
if (prevComponent) { ... }
```

`prevComponent` がある場合は `shouldUpdateReactComponent` に応じてアップデートするっぽい。 [→](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L472)

```js
var component = ReactMount._renderNewRootComponent(
  nextWrappedElement,
  container,
  shouldReuseMarkup,
  nextContext,
  callback
)._renderedComponent.getPublicInstance();
```

この時点で `component` は `<span data-reactroot="">hello</div>` となっている。 [→](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L528)  
`ReactMount._renderNewRootComponent()` は `TopLevelWrapper` のインスタンスを持ってるオブジェクトを返す、そこから `_._renderedComponent.getPublicInstance()` で `<span>hello</span>` を取り出している。

あとは `component` を返すだけ。

```js
return component;
```

### [ReactMount._renderNewRootComponent](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L356)

目的: 新しいコンポーネントを生成する。

```
ReactMount._renderNewRootComponent(
  nextElement, container, shouldReuseMarkup, context, callback
) |            |          |                  |        |
  |            div#app    null               {}       undefined
  TopLevelWrapper
```

```js
var componentInstance = instantiateReactComponent(nextElement, false);
```

`ReactCompositeComponentWrapper`を生成。 [→](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L382)  
この時点でまだ `<span/>` は生成されていない。  
`componentInstance._currentElement` が `TopLevelWrapper` となっている。

```js
ReactUpdates.batchedUpdates(
  batchedMountComponentIntoNode,
  componentInstance,
  container,
  shouldReuseMarkup,
  context
);
```

重要なのはこっち。 [→](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L394)  
`ReactUpdates.batchedUpdates()` はとりあえず第一引数の関数を残りの引数で呼び出すと読めば良い。

つまりこういうこと。

```js
ReactMount.batchedMountComponentIntoNode(
  componentInstance,
  container,
  shouldReuseMarkup,
  context
)
```

あとは `componentInstance` を返すだけ.

```js
return componentInstance;
```

---

### [ReactMount.batchedMountComponentIntoNode](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L142)

```
ReactMount.batchedMountComponentIntoNode(
  componentInstance, container, shouldReuseMarkup, context
) |                  |          |                  |
  |                  div#app    null               {}
  ReactCompositComponentWrapper { _currentElement: TopLevelWrapper }
```

```js
transaction.perform(
  mountComponentIntoNode,
  null,
  componentInstance,
  container,
  transaction,
  shouldReuseMarkup,
  context
);
```

`transaction.perform` も第一引数の関数を残りの引数で呼び出すやつ。  
transaction の準備? のあと `ReactMount.mountComponentIntoNode` を呼んでいる。 [→](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L152)

### [ReactMount.mountComponentIntoNode](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L94)

```
ReactMount.mountComponentIntoNode(
  wrapperInstance, container, transaction, shouldReuseMarkup, context
) |                |          |            |                  |
  |                |          |            null               {}
  |                div#app    ReactReconcileTransaction
  ReactCompositComponentWrapper { _currentElement: TopLevelWrapper }
```

```js
var markup = ReactReconciler.mountComponent(
  wrapperInstance,
  transaction,
  null,
  ReactDOMContainerInfo(wrapperInstance, container),
  context,
  0 /* parentDebugID */
);
```

ここで markup をつくる。 [→](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L112)  
この markup には `<span>hello</span>` が含まれている。

先に `ReactReconciler.mountComponent()` で markup を作るところを見ていく。  
そのあと作った markup をどうするか見る。 → :one:

---
## [ReactReconciler.js](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactReconciler.js)

### [ReactReconciler.mountComponent](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactReconciler.js#L40)

markup を作って返す。

```
ReactReconciler.mountComponent(
  internalInstance, transaction, hostParent, hostContainerInfo, context, parentDebugID
) |                 |            |           |                  |        |
  |                 |            null        object             {}       0
  |                 ReactReconcileTransaction
  ReactCompositComponentWrapper { _currentElement: TopLevelWrapper }
```

```js
var markup = internalInstance.mountComponent(
  transaction,
  hostParent,
  hostContainerInfo,
  context,
  parentDebugID
);
```

`internalInstance: ReactCompositeComponent` の `mountComponent` を呼び出す。  [→](https://github.com/facebook/react/blob/e43aaab2547870bc80c0b778604c5ee55b1d87f0/src/renderers/shared/stack/reconciler/ReactReconciler.js#L57)  
`markup.node` に `<span>hello</span>` が生成されている。

```js
return markup;
```

---

## [ReactCompositeComponent.js](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js)

### [ReactCompositeComponent#mountComponent](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L181)

長い。

- `ReactComponent` のインスタンスを生成する [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L201)
  - 今回は `TopLevelWrapper` のインスタンスが生成される
- `componentWillMount()` が定義されていれば実行 [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L336)
- markup を作る。 [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L353)
- `componentDidMount()` が定義されていれば実行 [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L372)
  - 直ちに実行するのではなくて transaction に enqueue する
- ペンディングされているコールバックを実行 [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L387)

```
ReactCompositeComponent#mountComponent(
  transaction, hostParent, hostContainerInfo, context
) |            |           |                  |
  |            null        object             {}
  ReactReconcileTransaction
```

```js
markup = this.performInitialMount(
  renderedElement,
  hostParent,
  hostContainerInfo,
  transaction,
  context
);
```

markup を作る。 [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L363)  
`markup.node` に `<span>hello</span>` が生成されている。

```js
return markup;
```

### [ReactCompositeComponent#performInitialMount](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L507)

```
ReactCompositeComponent#performInitialMount(
  renderedElement, hostParent, hostContainerInfo, transaction, context  
) |                |           |                  |            |
  |                |           |                  |             {}
  |                null        object             ReactReconcileTransaction
  undefined
```

```js
renderedElement = this._renderValidatedComponent();
// {"type":"span","key":null,"ref":null,"props":{"children":"hello"},"_owner":null,"_store":{}}
```

ここで `renderedElement` を作っている。  [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L516)  

`ReactCompositeComponent#_renderValidatedComponent()` で `TopLevelWrapper` の `render()` が呼ばれる。 [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L1153)  
（`TopLevelWrapper` の中身が `React.Component` の場合、そっちの `render()` とかはまだ呼び出されない）

```js
var child = this._instantiateReactComponent(
  renderedElement,
  nodeType !== ReactNodeTypes.EMPTY /* shouldHaveDebugID */
);
```

ここで `ReactDOMComponent` が生成される。 [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L521)

```js
var markup = ReactReconciler.mountComponent(
  child,
  transaction,
  hostParent,
  hostContainerInfo,
  this._processChildContext(context),
  debugID
);
```

さっき作った `ReactDOMComponent` の `mountComponent` が呼んで markup を作る。 [→](https://github.com/facebook/react/blob/7ef856aa3676f898ac5014f77a3e176bdf727350/src/renderers/shared/stack/reconciler/ReactCompositeComponent.js#L532)

```js
return markup;
```

---

## [ReactDOMComponent](https://github.com/facebook/react/blob/6beb87eb73cb7a5e390c47614b5f796afdfa3f65/src/renderers/dom/stack/client/ReactDOMComponent.js)

`ReactDOMComponent` は `ReactMultiChild` を継承している。`ReactMultiChild` は下記のような複数の子要素を処理するメソッドを持っている。

```html
<ul>
  <li>foo</li>
  <li>bar</li>
  <li>baz</li>
</ul>
```

### [ReactDOMComponent#mountComponent](https://github.com/facebook/react/blob/6beb87eb73cb7a5e390c47614b5f796afdfa3f65/src/renderers/dom/stack/client/ReactDOMComponent.js#L450)

```
ReactDOMComponent#mountComponent(
  transaction, hostParent, hostContainerInfo, context
) |            |           |                  |
  |            null        object             {}
  ReactReconcileTransaction
```

```js
el = ownerDocument.createElement(type);
```

ここで `HTMLSpanElement` が生成される。中身はまだ空っぽ。  [→](https://github.com/facebook/react/blob/6beb87eb73cb7a5e390c47614b5f796afdfa3f65/src/renderers/dom/stack/client/ReactDOMComponent.js#L569)

```js
var lazyTree = DOMLazyTree(el);
this._createInitialChildren(transaction, props, context, lazyTree);
mountImage = lazyTree;
```

ここでタグの中身を処理する。  [→](https://github.com/facebook/react/blob/6beb87eb73cb7a5e390c47614b5f796afdfa3f65/src/renderers/dom/stack/client/ReactDOMComponent.js#L595)

```js
return mountImage;
```

### [ReactDOMComponent#_createInitialChildren](https://github.com/facebook/react/blob/6beb87eb73cb7a5e390c47614b5f796afdfa3f65/src/renderers/dom/stack/client/ReactDOMComponent.js#L786)

```
ReactDOMComponent#_createInitialChildren(
  transaction, props, context, lazyTree
) |            |      |        |
  |            |      {}       DOMLazyTree
  |            {"children":"hello"}
  ReactReconcileTransaction
```

```js
DOMLazyTree.queueText(lazyTree, contentToUse);
```

子要素が文字列か数値の場合、ここで `HTMLSpanElement` の `textContent` に "hello" をセットしている。 [→](https://github.com/facebook/react/blob/6beb87eb73cb7a5e390c47614b5f796afdfa3f65/src/renderers/dom/stack/client/ReactDOMComponent.js#L807)

```js
var mountImages = this.mountChildren(
  childrenToUse,
  transaction,
  context
);
for (var i = 0; i < mountImages.length; i++) {
  DOMLazyTree.queueChild(lazyTree, mountImages[i]);
}
```

子要素が文字列でも数値でもない場合は、順番に `mountComponent()` して `DOMLazyTree.queueChild()` される。 [→](https://github.com/facebook/react/blob/6beb87eb73cb7a5e390c47614b5f796afdfa3f65/src/renderers/dom/stack/client/ReactDOMComponent.js#L810)

`mountComponent()` はここでやってる。  [ReactMultiChild#mountChildren](https://github.com/facebook/react/blob/f33f03e3572d11e6810f4ce110eb3af97cbd24a8/src/renderers/shared/stack/reconciler/ReactMultiChild.js#L248)

---

## [DOMLazyTree.js](https://github.com/facebook/react/blob/e3131c1d55d6695c2f0966379535f88b813f912b/src/renderers/dom/stack/client/DOMLazyTree.js)

DOMLazyTree はブラウザごとに異なるDOM操作のパフォーマンスを考慮している。  
See https://github.com/spicyj/innerhtml-vs-createelement-vs-clonenode.

---

:one: [`ReactMount.mountComponentIntoNode`](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L94) で markup を作ったその後。

```js
ReactMount._mountImageIntoNode(
  markup,
  container,
  wrapperInstance,
  shouldReuseMarkup,
  transaction
);
```

ここで `<span>hello</span>` を `div#app` に追加している。  [→](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L126)

### [ReactMount._mountImageIntoNode](https://github.com/facebook/react/blob/cb6da8e922c426dfae11a5da159ff9e8e176ce59/src/renderers/dom/stack/client/ReactMount.js#L628)

```
ReactMount._mountImageIntoNode(
  markup, container, instance, shouldReuseMarkup, transaction
) |       |          |         |                  |
  |       |          |         null               ReactReconcileTransaction
  |       div#app    ReactCompositeComponentWrapper
  object { node: HTMLSpanElement }
```

```js
DOMLazyTree.insertTreeBefore(container, markup, null);
```

DOM操作の現場。 [→](https://github.com/facebook/react/blob/e3131c1d55d6695c2f0966379535f88b813f912b/src/renderers/dom/stack/client/DOMLazyTree.js#L60)

---

## 所感

- レンダリングの大まかな流れは見えたように思う
- 初回のレンダリングだけでは Virtual DOM の強さみたいなのは見えない
- トランザクションの部分は全然見てなかったけど多分好きなやつな気がする
