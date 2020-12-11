#### React.Children 常用 API 源码解读---基于 react17.0.1

我们在写 React 组件时候，经常会有通过 this.props.children 传递给组件进行渲染包裹页面，通常会通过 this.props.children.map()来遍历渲染，其实 React 有提供一个 API 来做这里。也就是我们今天要说的 React.Children.map

###### 问题

1. 直接数组的 map 和 React.Children.map 有什么区别
2. React.Children.map 和 React.Children.forEach 有什么区别

首先，我们写个例子

```
import React from "react";
import "./App.css";

function ComWarp(props) {
  return (
    <div>
      {React.Children.map(props.children, (c) => (
        <div className="bg">{c}</div>
      ))}
    </div>
  );
}

function App() {
  return (
    <div className="App">
      <ComWarp>
        <div>1</div>
        <div>2</div>
      </ComWarp>
    </div>
  );
}

export default App;

```

上面写了组件 ComWarp,接受 props.children 来进行处理

```
var Children = {
  map: mapChildren,
  forEach: forEachChildren,
  count: countChildren,
  toArray: toArray,
  only: onlyChild
};
```

React 的 Children 是个对象，里面有 map、forEach 等等方法，map 对应的就是 mapChildren 函数，我们看看这个函数做了什么

```
function mapChildren(children, func, context) {
  if (children == null) {
    return children;
  }

  var result = [];
  var count = 0;
  mapIntoArray(children, result, '', '', function (child) {
    return func.call(context, child, count++);
  });
  return result;
}
```

1. children - 我们传递进来的 ComWarp 包裹的子内容是个数组，里面是每个 div 的对象（\$\$type 是 react.element）
2. 创建了一个空的数组 result 最后返回，我们继续看看 mapIntoArray 函数,这个函数比较长，我们分几个部分来说

```
function mapIntoArray(children, array, escapedPrefix, nameSoFar, callback) {
    return subtreeCount;
}

第一部分

var type = typeof children;

if (type === 'undefined' || type === 'boolean') {
// All of the above are perceived as null.
children = null;
}

var invokeCallback = false;

if (children === null) {
    invokeCallback = true;
} else {
    switch (type) {
      case 'string':
      case 'number':
        invokeCallback = true;
        break;

      case 'object':
        switch (children.$$typeof) {
          case REACT_ELEMENT_TYPE:
          case REACT_PORTAL_TYPE:
            invokeCallback = true;
        }

    }
}

```

第一部分是判断传入的 children 的类型，上面说了 children 是一个数组，那也就是 object 类型，走下 switch object,发现没有 case 符合，因为只是个纯数组而已没有\$\$typeof，他的子级才会有

```
  var child;
  var nextName;
  var subtreeCount = 0; // Count of children found in the current subtree.
  var nextNamePrefix = nameSoFar === '' ? SEPARATOR : nameSoFar + SUBSEPARATOR;
  if (Array.isArray(children)) {
    for (var i = 0; i < children.length; i++) {
      child = children[i];
      nextName = nextNamePrefix + getElementKey(child, i);
      subtreeCount += mapIntoArray(child, array, escapedPrefix, nextName, callback);
    }
  } else {
    var iteratorFn = getIteratorFn(children);
    if (typeof iteratorFn === 'function') {
      var iterableChildren = children;
      {
        // Warn about using Maps as children
        if (iteratorFn === iterableChildren.entries) {
          if (!didWarnAboutMaps) {
            warn('Using Maps as children is not supported. ' + 'Use an array of keyed ReactElements instead.');
          }

          didWarnAboutMaps = true;
        }
      }
      var iterator = iteratorFn.call(iterableChildren);
      var step;
      var ii = 0;

      while (!(step = iterator.next()).done) {
        child = step.value;
        nextName = nextNamePrefix + getElementKey(child, ii++);
        subtreeCount += mapIntoArray(child, array, escapedPrefix, nextName, callback);
      }
    } else if (type === 'object') {
      var childrenString = '' + children;
      {
        {
          throw Error( "Objects are not valid as a React child (found: " + (childrenString === '[object Object]' ? 'object with keys {' + Object.keys(children).join(', ') + '}' : childrenString) + "). If you meant to render a collection of children, use an array instead." );
        }
      }
    }
  }
```

这里其实就会判断 children 是个数组的话(成立)，就会遍历 children，得到每个子级继续进行 mapIntoArray(看到没递归函数)

nextName 定义一个常量+本身的 key,用于动态生成 key 给没个元素的 props 添加后续方法 cloneAndReplaceKey

i=0,child 的子级就是具体某一个 react 元素，继续执行 mapIntoArray,此时我们再返回到第一部分，子级就是一个 object,且是个 react 元素\$\$typeof 是 REACT_ELEMENT_TYPE, invokeCallback 设置 true,我们继续看 true 后会执行什么

```
 if (invokeCallback) {
    var _child = children;
    var mappedChild = callback(_child); // If it's the only child, treat the name as if it was wrapped in an array
    // so that it's consistent if the number of children grows:

    var childKey = nameSoFar === '' ? SEPARATOR + getElementKey(_child, 0) : nameSoFar;

    if (Array.isArray(mappedChild)) {
      var escapedChildKey = '';

      if (childKey != null) {
        escapedChildKey = escapeUserProvidedKey(childKey) + '/';
      }

      mapIntoArray(mappedChild, array, escapedChildKey, '', function (c) {
        return c;
      });
    } else if (mappedChild != null) {
      if (isValidElement(mappedChild)) {
        mappedChild = cloneAndReplaceKey(mappedChild, // Keep both the (mapped) and old keys if they differ, just as
        // traverseAllChildren used to do for objects as children
        escapedPrefix + ( // $FlowFixMe Flow incorrectly thinks React.Portal doesn't have a key
        mappedChild.key && (!_child || _child.key !== mappedChild.key) ? // $FlowFixMe Flow incorrectly thinks existing element's key can be a number
        escapeUserProvidedKey('' + mappedChild.key) + '/' : '') + childKey);
      }

      array.push(mappedChild);
    }

    return 1;
  }
```

执行了我们传递过来的 callback 函数也就是(c) => ()),执行完后判断返回的 mappedChild 的类型，如果不是数组且不是 null,就会把 mappedChild 放入到 array（一开始我们定义的空数组）

如果 mappedChild 是数组，也就是我们传递的(c) =>[c,c],那就会继续 mapIntoArray，此时 mapIntoArray 的 chilren 是返回的[c，c]的数组，callback 改造成 c=>c，而不是开始的(c) =>[c,c]

最后往复递归形成新的一个数组 result

### 流程图

![image](img/childrenmap.png)

---

##### React.Children.forEach

```
function forEachChildren(children, forEachFunc, forEachContext) {
  mapChildren(children, function () {
    forEachFunc.apply(this, arguments); // Don't return anything.
  }, forEachContext);
}

```

调用的也是 mapChildren 函数，区别于 map 是 map 会返回一个新的数组，而 forEach 只是处理 forEachFunc 不返回

---

##### toArray

```
function toArray(children) {
  return mapChildren(children, function (child) {
    return child;
  }) || [];
}
```

调用 mapChildren，返回处理好的数组

---

##### onlyChild 返回 react 元素

```
function onlyChild(children) {
  if (!isValidElement(children)) {
    {
      throw Error( "React.Children.only expected to receive a single React element child." );
    }
  }

  return children;
}

function isValidElement(object) {
  return typeof object === 'object' && object !== null && object.$$typeof === REACT_ELEMENT_TYPE;
}

```

如果不是 react 的元素，就抛出异常，是直接返回 react 元素
