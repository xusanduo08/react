解决 'import type' declarations can only be used in TypeScript files. 问题：
1. 先安装扩展 flow language support
2. 修改设置 "javascript.validate.enable": false

```javascript
import React from 'react';
import ReactDOM from 'react-dom';

ReactDOM.render(reactElement, container)
```
render方法的调用连：
ReactDOM.render(): ReactDOMLegacy.js

```javascript
export function render(
  element: React$Element<any>,
  container: Container,
  callback: ?Function,
) {
  invariant(
    isValidContainer(container),
    'Target container is not a DOM element.',
  );
  
  return legacyRenderSubtreeIntoContainer(
    null,
    element,
    container,
    false,
    callback,
  );
}
```

legacyRenderSubtreeIntoContainer():

```javascript
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: Container,
  forceHydrate: boolean,
  callback: ?Function,
) {
  // TODO: Without `any` type, Flow says "Property cannot be accessed on any
  // member of intersection type." Whyyyyyy.
  let root: RootType = (container._reactRootContainer: any);
  let fiberRoot;
  if (!root) {
    // Initial mount
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Initial mount should not be batched.
    // 第一次渲染时走的是这个逻辑
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Update
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```

unbatchedUpdates()方法比较建单：对于初次渲染，可直接理解为执行入参函数

```js
export function unbatchedUpdates<A, R>(fn: (a: A) => R, a: A): R {
  const prevExecutionContext = executionContext;
  executionContext &= ~BatchedContext;
  executionContext |= LegacyUnbatchedContext;
  try {
    return fn(a); // 《=======这执行updateContainer()
  } finally {
    executionContext = prevExecutionContext;
    if (executionContext === NoContext) {
      // Flush the immediate callbacks that were scheduled during this batch
      flushSyncCallbackQueue();
    }
  }
}
```

updateContainer():

```js
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): ExpirationTime {

  const current = container.current;
  const currentTime = requestCurrentTimeForUpdate();

  const suspenseConfig = requestCurrentSuspenseConfig();
  const expirationTime = computeExpirationForFiber(
    currentTime,
    current,
    suspenseConfig,
  );

  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }


  const update = createUpdate(expirationTime, suspenseConfig);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {

    update.callback = callback;
  }

  enqueueUpdate(current, update);
  scheduleUpdateOnFiber(current, expirationTime);

  return expirationTime;
}
```
enqueueUpdate从字面可以理解为将这次更新放置在队列中，然后由scheduleUpdateOnFiber来调度这次更新
