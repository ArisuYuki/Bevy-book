## 3.1 全局单例

​	在第一章中写到`Resource`是一个全局单例。用于保存一些在游戏的整个生命周期里都存在的数据，例如游戏设置等。

​	要创建一个`Resource`，只需要使用`Resource`指令即可，然后我们便可以声明一个单例并初始化。

```rust
#[derive(Resource,Default)]
struct Setting{
  source:f32
};

//在App中直接初始化，在这里我们实现了Default，因此可以只指定类型
App.insert_resource::<Setting>()

//或者使用commands动态的添加和删除
fn add_score(mut commands: Commands) {
  commands.init_resource::<Setting>();
  //或者我们也可以在这里删除一些资产
  commands.remove_resource::<Setting>();
}

```

​	在使用时，只需要在需要使用的系统上使用`Res`或者`ResMut`来指定资源的类型即可。

```rust
//获得资产的可变引用以便更改
fn some_system(mut score: ResMut<Score>) 
//只获得共享引用
fn some_system(score: Res<Score>) 
//如果资产可能尚未创建，那么需要使用Option使之变为可选
fn some_system(mut score: Option<ResMut<Score>>) 
```

​	除了作为一个可以在系统中共享的数据单例，Bevy中许多功能的实现也都是基于`Resource`来实现的，在前面我们能已经介绍了一部分，这些内容如下。

```rust
Res<Time> //自应用启动以来的时间，以及上一帧逝去的时间
Res<Events<E>> //用于访问各种引擎事件
Res<Assets<T>> // 用于加载静态资产
Res<Window> //存储主窗口的属性
Res<ButtonInput<B>> //用于查询键盘或者鼠标的状态
```

## 3.2 Resource的其他用法

​	`Resource`的功能虽然简单，但是能实现的功能是相当强大的。这里，我们将介绍几个利用`Resource`可以方便实现的事。

​	在第二章中的`State`一节中，我们利用不同的`State`和`run_if`方法来动态的决定某些系统的运行，但当我们需要根据更复杂的逻辑来实现动态决定系统的运行状态时，我们该怎么做？

​	`run_if`方法可以接受一个可有查询参数的返回布尔类型的闭包作为参数，这意味着我们可以利用查询系统来结合`Resource`进行判断，就像下面这样。通过这种方法，可以结合各种`Resource`来动态的决定系统的运行状态。

```rust
some_system
	.run_if(|counter: Res<InputCounter>| counter.is_changed() && !counter.is_added())
```

​	
