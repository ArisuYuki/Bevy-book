## 4.1 资产加载与保存

​	在 Bevy 引擎中，`AssetReader`、`AssetLoader`、`AssetSaver` 和 `AssetWriter` 都是资产系统的重要组成部分，它们各自承担着不同的职责。

### 4.1.1 AssetLoader

​	`AssetLoader`是我们主要用于加载资产的核心类型，在第一章的自定义资产中，我们做了如下几件事情：

1. 使用`Asset`指令定义一个`PointCloud`资产
2. 定义对应的`LasLoader`、`LasLoaderError`、`LasLoaderSettings`
3. 为`LasLoader`实现异步的`load`方法，并从中返回一个`Asset`

​	通过这些操作，我们便可以在系统中这样使用我们自定义的加载器以方便的来实现自定义类型的文件加载，并且，我们还可以通过`LasLoaderSettings`来控制加载的行为。

```rust
fn main() {
  App::new()
    .add_plugins(DefaultPlugins)
  	//通过这两个方法注册相应的加载器和资产类型
    .init_asset_loader::<LasLoader>()
    .init_asset::<PointCloud>()
  	.add_systems(Startup, load_pointcloud)
    .run();
}

//现在，我们应该能够直接使用这些资产类型了
fn load_pointcloud(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
){
  //我们不再简单的使用load方法，而是使用load_with_settings方法来指定setting参数
  let point_cloud_handler = asset_server.load_with_settings::<PointCloud>(
    "pointCloud.las"，
    |settings: &mut LasLoaderSettings| {
         //我们可以对这个默认对象做一些更改
    });
}
```

### 4.1.2 AssetReader

​	在为`LasLoader`实现异步的`load`方法时，你可能已经注意到了有一个`Reader`参数，他是一个类型擦除的`bevy_asset::io::Reader`特型对象，用于异步地将字节数据读取到缓冲区中。而这个特型对象，正是`AssetReader`的`read`方法返回的。

```rust
impl AssetLoader for LasLoader {
    type Asset = PointCloud;
    type Settings = LasLoaderSettings;
    type Error = LasLoaderError;

    async fn load(
        &self,
        reader: &mut dyn bevy_asset::io::Reader,
        _settings: &Self::Settings,
        _load_context: &mut LoadContext<'_>,
    ) -> Result<PointCloud, Self::Error> {
      let mut bin_data = Vec::new();
      reader.read_to_end(&mut bin_data).await?;
      //在这里编写真正加载数据的逻辑
      //let points = ..... 
      //然后返回一个资产
      Ok(PointCloud { points })
  }
```

​	`AssetReader`和`AssetLoader`都用于资产的加载，但是他们负责的层级是完全不同的。**`AssetReader`为不同的平台提供了一个统一的、异步的加载方式，使得我们的游戏能够在每个平台上以相同的方式读取资产；而`AssetLoader`负责将数据解析成正确的格式，以供程序的正确使用。**

​	大多数情况下，我们可以直接使用内置的`AssetReader`不需要对其进行自定义，但如果你需要在这个环节进行一些操作，那么Bevy也提供了相应的方法。详细的细节可以查看[文档](https://doc.qu1x.dev/bevy_trackball/bevy_asset/io/trait.AssetReader.html#tymethod.read)和[示例](https://github.com/bevyengine/bevy/blob/main/examples/asset/custom_asset_reader.rs)。

### 4.1.3 AssetSaver与AssetWriter

​	理解了`AssetReader`与`AssetLoader`的关系，我们可以猜到应该还有两个用于保存资产到本地的类型，这些类型就是`AssetWriter`与`AssetSaver`，他们的关系是类似的：`AssetWriter`提供了一个统一的、异步的写入方式，将我们的资产写入到文件系统中，而`AssetSaver`则负责将数据转换成需要保存的格式。二者的具体使用方法可以查看[文档](https://doc.qu1x.dev/bevy_trackball/bevy_asset/saver/trait.AssetSaver.html)，需要为`AssetSaver`实现一个`save`方法，该方法和`load`非常相似，这里不再赘述。

### 4.1.4 Embedded Asset 与Internal Asset

​	有些时候，我们希望将资产打包进入二进制程序中，然后在程序中直接读取这些资产而不是从硬盘里加载。例如，我们可能编写了一些wgsl着色器而又不想将这些代码作为文件存储在磁盘里，这时就需要将其直接内嵌在二进制的程序中，不过我们会需要一些额外的手段来告诉Bevy如何读取这些内嵌的资源。

​	Bevy中使用宏来嵌入资产到程序中并读取，他们是`embedded_asset!`宏和`load_embedded_asset!`宏。

​	`embedded_asset!`宏接受两个或三个参数，当接受两个参数的时候，第一个是当前`app`的可变引用，第二个是需要内嵌的资源的相对于当前目录的路径。假设我们的项目目录如下，我们现在想要将rock.wgsl内嵌到程序中。

```tex
bevy_rock
├── src
│   ├── render
│   │   ├── rock.wgsl
│   │   └── mod.rs
│   └── lib.rs
└── Cargo.toml
```

​	在render目录下的mod.rs中，我们可以编写一个插件，然后这样使用`embedded_asset!`宏和`load_embedded_asset!`宏。当使用宏加载后，文件将会位于一个虚拟的目录下，在这里是`embedded://bevy_rock/render/rock.wgsl`（注意到src路径已经被删除）。其中前面的`embedded`被称为`AssetSourceId`，每一类`AssetSourceId`都映射着对应的`AssetReader`和`AssetWriter`（从程序中之间内嵌的数据读取方式和从磁盘的读取方式不同，所以需要保留这个信息告诉Bevy正确的读取方式）。

​	我们还可以利用`app`上的`register_asset_source`方法注册自己的`AssetSourceId`，将其绑定到某些文件夹以便在加载资产时通过前戳来快速访问资产，详细的方法可以参考[文档](https://doc.qu1x.dev/bevy_trackball/bevy_asset/trait.AssetApp.html#tymethod.register_asset_source)，这里不再赘述。

```rust
//使用两个参数的版本时，第一个参数是app的可变引用，第二个是资产的路径（相对当前目录）
embedded_asset!(app, "rock.wgsl")
//使用三个参数的版本时，第三个参数是资产的路径，第二个参数是需要移除的路径部分
//因此embedded_asset!(app, "rock.wgsl")和embedded_asset!(app, "/src/", "rock.wgsl")是等效的
embedded_asset!(app, "/examples/rock_stuff/", "rock.wgsl")
```

​	然后，我们有两种方式可以加载这个资源。

```rust
//如果在当前的模块中加载，我们可以直接使用load_embedded_asset
let shader = load_embedded_asset!(&asset_server, "rock.wgsl");

//如果在其他模块中，我们可以使用asset_server和路径全称
let shader = asset_server.load::<Shader>("embedded://bevy_rock/render/rock.wgsl");
```

​	在Bevy 0.12之前，你可能会看到名为`load_internal_asset!`的宏，该宏的作用和上面是一样的，不过目前已经被`embedded_asset!`取代，因此不建议继续使用。

## 4.2 跟踪加载进度

​	复习一下在第一章中我们介绍的使用资产的方法，我们可以使用资产的句柄和`asset_server`来轮询资产的加载状态。

```rust
fn on_asset_event(
  mut commands: Commands,
  asset_server: Res<AssetServer>,
  share_image: Res<ShareImage>,
) {
  match asset_server.get_load_state(&share_image.handle) {
    Some(LoadState::NotLoaded) => {}
    Some(LoadState::Loading) => {}
    Some(LoadState::Loaded) => {
      //在这里使用handle，这时已经加载完成
    }
}
```

​	

## 4.2 Mesh与StandardMaterial