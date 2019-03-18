---
id: direct-manipulation
title: 直接操作
---

有时候我们需要直接改动组件并触发局部的刷新，但不使用 state 或是 props。譬如在浏览器中使用 React 库，有时候会需要直接修改一个 DOM 节点，而在手机 App 中操作 View 时也会碰到同样的情况。在 React Native 中，`setNativeProps`就是等价于直接操作 DOM 节点的方法。

> 什么时候使用 setNativeProps 呢？在（不得不）频繁刷新而又遇到了性能瓶颈的时候。
>
> 直接操作组件并不是应该经常使用的工具。一般来说只是用来创建连续的动画，同时避免渲染组件结构和同步太多视图变化所带来的大量开销。`setNativeProps`是一个“简单粗暴”的方法，它直接在底层（DOM、UIView 等）而不是 React 组件中记录 state，这样会使代码逻辑难以理清。所以在使用这个方法之前，请尽量先尝试用`setState`和[shouldComponentUpdate](http://facebook.github.io/react/docs/advanced-performance.html#shouldcomponentupdate-in-action)方法来解决问题。

## setNativeProps 与 TouchableOpacity

[TouchableOpacity](https://github.com/facebook/react-native/blob/master/Libraries/Components/Touchable/TouchableOpacity.js)这个组件就在内部使用了`setNativeProps`方法来更新其子组件的透明度：

```javascript
setOpacityTo(value) {
  // Redacted: animation related code
  this.refs[CHILD_REF].setNativeProps({
    opacity: value
  });
},
```

由此我们可以写出下面这样的代码：子组件可以响应点击事件，更改自己的透明度。而子组件自身并不需要处理这件事情，也不需要在实现中做任何修改。

```javascript
<TouchableOpacity onPress={this._handlePress}>
  <View style={styles.button}>
    <Text>Press me!</Text>
  </View>
</TouchableOpacity>
```

如果不使用`setNativeProps`来实现这一需求，那么一种可能的办法是把透明值保存到 state 中，然后在`onPress`事件触发时更新这个值：

```javascript
constructor(props) {
  super(props);
  this.state = { myButtonOpacity: 1, };
}

render() {
  return (
    <TouchableOpacity onPress={() => this.setState({myButtonOpacity: 0.5})}
                      onPressOut={() => this.setState({myButtonOpacity: 1})}>
      <View style={[styles.button, {opacity: this.state.myButtonOpacity}]}>
        <Text>Press me!</Text>
      </View>
    </TouchableOpacity>
  )
}
```

比起之前的例子，这一做法会消耗大量的计算 —— 每一次透明值变更的时候 React 都要重新渲染组件结构，即便视图的其他属性和子组件并没有变化。一般来说这一开销也不足为虑，但当执行连续的动画以及响应用户手势的时候，只有正确地优化组件才能提高动画的流畅度。

如果你看过[NativeMethodsMixin](https://github.com/facebook/react-native/blob/master/Libraries/Renderer/oss/ReactNativeRenderer-prod.js)中的`setNativeProps`方法的实现，你就会发现它实际是对`RCTUIManager.updateView`的封装 —— 而这正是重渲染所触发的函数调用，具体可以参看[ReactNativeBaseComponent.js 中的 receiveComponent](https://github.com/facebook/react/blob/master/src/renderers/native/ReactNativeBaseComponent.js).

## 复合组件与 setNativeProps

复合组件并不是单纯的由一个原生视图构成，所以你不能对其直接使用`setNativeProps`。比如下面这个例子：

```SnackPlayer name=setNativeProps%20with%20Composite%20Components
import React from 'react';
import { Text, TouchableOpacity, View } from 'react-native';

class MyButton extends React.Component {
  render() {
    return (
      <View>
        <Text>{this.props.label}</Text>
      </View>
    )
  }
}

export default class App extends React.Component {
  render() {
    return (
      <TouchableOpacity>
        <MyButton label="Press me!" />
      </TouchableOpacity>
    )
  }
}
```

跑这个例子会马上看到一行报错： `Touchable child must either be native or forward setNativeProps to a native component`。这是因为`MyButton`并非直接由原生视图构成，而我们只能给原生视图设置透明值。你可以尝试这样去理解：如果你通过`React.createClass`方法自定义了一个组件，直接给它设置样式 prop 是不会生效的，你得把样式 props 层层向下传递给子组件，直到子组件是一个能够直接定义样式的原生组件。同理，我们也需要把`setNativeProps`传递给由原生组件封装的子组件。

#### 将 setNativeProps 传递给子组件

具体要做的就是在我们的自定义组件中再封装一个`setNativeProps`方法，其内容为对合适的子组件调用真正的`setNativeProps`方法，并传递要设置的参数。

```SnackPlayer name=Forwarding%20setNativeProps
import React from 'react';
import { Text, TouchableOpacity, View } from 'react-native';

class MyButton extends React.Component {
  setNativeProps = (nativeProps) => {
    this._root.setNativeProps(nativeProps);
  }

  render() {
    return (
      <View ref={component => this._root = component} {...this.props}>
        <Text>{this.props.label}</Text>
      </View>
    )
  }
}

export default class App extends React.Component {
  render() {
    return (
      <TouchableOpacity>
        <MyButton label="Press me!" />
      </TouchableOpacity>
    )
  }
}
```

现在你可以用`MyButton`来代替`TouchableOpacity`了！有一点需要特别说明：这里我们使用了[ref 回调](https://doc.react-china.org/docs/refs-and-the-dom.html)语法，而不是传统的字符串型 ref 引用。

你可能还会注意到我们在向下传递 props 时使用了`{...this.props}`语法（这一用法的说明请参考[对象的扩展运算符](http://es6.ruanyifeng.com/#docs/object)）。这是因为`TouchableOpacity`本身其实也是个复合组件， 它除了要求在子组件上执行`setNativeProps` 以外，还要求子组件对触摸事件进行处理。因此，它会传递多个 props，其中包含了[onmoveshouldsetresponder](view.html#onmoveshouldsetresponder) 函数，这个函数需要回调给`TouchableOpacity`组件，以完成触摸事件的处理。与之相对的是`TouchableHighlight`组件，它本身是由原生视图构成，因而只需要我们实现`setNativeProps`。

## setNativeProps to clear TextInput value

清除文本输入框内容是 setNativeProps 的另一个常用方式.当 bufferDely 很低,用户输入很快时,文本输入框的 controlled参数有时会丢失. 一些开发者偏向于跳过prop操作,直接 使用 setNativeProps 来进行文本输入框值的显示操作.下面就是一个按下按钮清除文本输入框内容的例子:

```SnackPlayer name=Clear%20text
import React from 'react';
import { TextInput, Text, TouchableOpacity, View } from 'react-native';

export default class App extends React.Component {
  clearText = () => {
    this._textInput.setNativeProps({text: ''});
  }

  render() {
    return (
      <View style={{flex: 1}}>
        <TextInput
          ref={component => this._textInput = component}
          style={{height: 50, flex: 1, marginHorizontal: 20, borderWidth: 1, borderColor: '#ccc'}}
        />
        <TouchableOpacity onPress={this.clearText}>
          <Text>Clear text</Text>
        </TouchableOpacity>
      </View>
    );
  }
}
```

# 避免和 render 方法的冲突

如果要更新一个由 render 方法来维护的属性，则可能会碰到一些出人意料的 bug。因为每一次组件重新渲染都可能引起属性变化，这样一来，之前通过`setNativeProps`所设定的值就被完全忽略和覆盖掉了。

## setNativeProps 与 shouldComponentUpdate

通过[巧妙运用`shouldComponentUpdate`方法](https://facebook.github.io/react/docs/advanced-performance.html#avoiding-reconciling-the-dom)，可以避免重新渲染那些实际没有变化的子组件所带来的额外开销，此时使用`setState`的性能已经可以与`setNativeProps`相媲美了。

## 其他的原生操作

此处描述的方法大多数React Native 已经提供. 但是 这些在组合组件是不支持的,因为原生的视图不提供支持,包含你自己的应用中你自定的绝大多数组件

### measure(callback)

量测视图在屏幕上的坐标,宽度,高度;异步回调这些参数.如果量测成功,回调如下参数:

- x
- y
- width
- height
- pageX
- pageY

注意,这些参数必须在视图组件在本地渲染完成后才能返回视图视图量测值.若你想尽快的获取视图组件的量测值,可考虑使用[`onLayout` prop](view.md#onlayout) 来实现.

### measureInWindow(callback)

获取指定视图在窗口上的位置,异步返回量测值. 如果根视图在另一个视图上,将返回绝对的位置. 获取成功返回如下参数:

- x
- y
- width
- height

### measureLayout(relativeToNativeNode, onSuccess, onFail)

例如 `measure()`方法, 量测相对于祖视图的位置关系, 特别的是 `relativeToNativeNode`. 意味着返回的是相对于租视图原点的x,y.

 通常我们使用 `findNodeHandle(component)`方法获取组件中原生的节点 .

```javascript
import {findNodeHandle} from 'react-native';
```

### focus()

获取给定输入框或者视图的焦点,即第一响应状态.更进一步的变换操作依赖与不同的平台和不同的视图.

### blur()

与 `focus()` 方法相反,该方法用来给定视图或者输入框的焦点,即取消给定视图或输入框的第一响应状态.
