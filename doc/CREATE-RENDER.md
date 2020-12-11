#### React.createElement --- 基于 react17.0.1

在写 React,我们经常创建一个 jsx 文件，在里面编写代码就可以了，我们看看 JSX 转换后的代码

![babel把JSX转换成的代码](/img/create1.png)

实际调用的就是 React.createElement，直接看里面做了什么

```

第一部分

export function createElement(type, config, children) {
  let propName;

  // Reserved names are extracted
  const props = {};

  let key = null;
  let ref = null;
  let self = null;
  let source = null;

  if (config != null) {
    if (hasValidRef(config)) {
      ref = config.ref;
      if (__DEV__) {
        warnIfStringRefCannotBeAutoConverted(config);
      }
    }
    if (hasValidKey(config)) {
      key = '' + config.key;
    }

    self = config.__self === undefined ? null : config.__self;
    source = config.__source === undefined ? null : config.__source;
    // Remaining properties are added to a new props object
    for (propName in config) {
      if (
        hasOwnProperty.call(config, propName) &&
        !RESERVED_PROPS.hasOwnProperty(propName)
      ) {
        props[propName] = config[propName];
      }
    }
  }

}
```

1.定义了一些 props，key、ref、self、source 变量，判断 config(就是元素上的属性配置对象)有的话进行处理
a.赋值我们常用的 ref 和 key，还有些 self 和 source
b.遍历 config 对象，把除了 key ref **self **source 都放入到定义的 props 对象里；这也是我们经常为什么开发中 props 不能传递的原因，key ref 关键字会单独处理不会到 props 里

```
下面代码

const childrenLength = arguments.length - 2;
  if (childrenLength === 1) {
    props.children = children;
  } else if (childrenLength > 1) {
    const childArray = Array(childrenLength);
    for (let i = 0; i < childrenLength; i++) {
      childArray[i] = arguments[i + 2];
    }
    if (__DEV__) {
      if (Object.freeze) {
        Object.freeze(childArray);
      }
    }
    props.children = childArray;
  }

  // Resolve default props
  if (type && type.defaultProps) {
    const defaultProps = type.defaultProps;
    for (propName in defaultProps) {
      if (props[propName] === undefined) {
        props[propName] = defaultProps[propName];
      }
    }
  }
  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props,
  );
```

1、获取 createElement 传递的参数 arguments，前面两个 type, config 去掉，获取后面多个 children 参数

2、如果只有一个 children 给 props 设置 children，有多个 children,遍历放入到 childArray 中，最后给 props 设置 children

3、找到组件里的 defaultProps 属性，开发中我们经常定义一个静态的 defaultProps 来给默认的 props 初始值，这里判断后，找到里面定义的给 props 传递的初始化值

4、最后返回一个 ReactElement 函数

```
const ReactElement = function(type, key, ref, self, source, owner, props) {
    const element = {
    // This tag allows us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,

    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,

    // Record the component responsible for creating this element.
    _owner: owner,
  };

  return element;
}
```

1.创建一个 element 对象，定义\$\$typeof 为 react 的 element-type 类型，这个定义在后续的 fiber.合成事件、更新中都会用到这个值判断

2.返回这个对象。

综上所述，其实 createElement 就是创建了 props 对象并赋值且单独处理的 ref、key 等，最后返回一个 type 为 REACT_ELEMENT_TYP 的 element 对象
