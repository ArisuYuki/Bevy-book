// TODO

要播放动画，不管怎样，我们都要先拿到模型，**Gltf模型其实是一种Asset**，因此我们在第三章中讨论的如何加载资产的那一整套，都可以完美的平移过来。

首先我们需要一个`Resource`来存储句柄，以便在不同的系统之间传递。当然也可以传递一些别的东西，不过目前而言我们主要关心的是其中的`Handle`。

```rust
#[derive(Resource)]
struct Fox(Handle<Gltf>);
```

然后，在`startup`阶段我们的`setup`系统中，只需要直接加载即可。

```rust
fn setup(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
    mut meshes: ResMut<Assets<Mesh>>,
    mut materials: ResMut<Assets<StandardMaterial>>,
) {
    commands.insert_resource(Fox(asset_server.load(FOX_PATH)));
    //....
}
```

在这里，我们将用上我们在第2章中介绍的条件运行(参见2.3.2 run_if)。我们希望在资源加载完成并且配置完成之前，不要运行我们的动画系统。（或者，我们也可以采用state的方法来切换系统的运行，你能想到怎么做吗？）

```rust
// 存储动画相关的资源
#[derive(Resource)]
struct Animations {
    // 至于AnimationNodeIndex和AnimationGraph是什么，等待下文我们揭晓
    animations: Vec<AnimationNodeIndex>,
    graph_handle: Handle<AnimationGraph>,
}

// 这个系统在资产加载完成之前将会一直尝试运行
app.add_systems(
    Update,
    spawn_fox_asset_when_ready.run_if(not(resource_exists::<Animations>)),
)// 这个系统只有在加载完成后运行
.add_systems(
    Update,
    keyboard_control.run_if(resource_exists::<Animations>),
)
```

在`spawn_fox_asset_when_ready`系统中，我们依然采用我们的三板斧。使用`is_loaded_with_dependencies`来判断是否已经加载完成(详见3.1.1节，要时常复习才能温故知新！)

```rust
fn spawn_fox_asset_when_ready(
    mut commands: Commands,
    fox_handle: Res<Fox>,
    asset_server: Res<AssetServer>,
    gltfs: Res<Assets<Gltf>>,
    mut graphs: ResMut<Assets<AnimationGraph>>,
) {
    // 如果尚未加载完成，那么先不执行
    if !asset_server.is_loaded_with_dependencies(&fox_handle.0) {
        return;
    }
	// 然后后利用句柄来获得我们的资产，你还记得吗？
    let fox = gltfs
        .get(&fox_handle.0)
        .expect("a loaded asset should exist in the glTF assets collection");

    // 很快你就会知道这里的三个动画对应了什么了
    // 现在，只需要知道我们有了一个AnimationGraph类型的graph
    // 还有了一个Vec<NodeIndex>类型的node_indices
    let (graph, node_indices) = AnimationGraph::from_clips([
        fox.named_animations["Run"].clone(),
        fox.named_animations["Walk"].clone(),
        fox.named_animations["Survey"].clone(),
    ]);

    // 保存这些动画信息，以便我们后续使用
    let graph_handle = graphs.add(graph);
    commands.insert_resource(Animations {
        animations: node_indices,
        graph_handle,
    });

    // 现在，我们才向bevy中添加真正的模型
    commands
        .spawn(SceneRoot(
            fox.default_scene
                .clone()
                .expect("a default scene exists in this file"),
        ))
    	// 这里用到了observe，还记得observe有什么用吗？（详见2.7.1节，再次强调温故知新的重要性～）
        .observe(setup_scene);
}

// 现在，每当SceneInstanceReady触发时，该函数将会被执行
// 可是SceneInstanceReady是什么？他的完整路径是bevy::scene::SceneInstanceReady
// 简而言之，这是bevy的一个内置事件，其定义如下，每当整个场景加载完成之后会被触发
// pub struct SceneInstanceReady {
//     pub entity: Entity,
//     pub instance_id: InstanceId,
// }
// 而AnimationPlayer组件，是跟随scene自动添加的
// 所以当SceneInstanceReady触发的时候，AnimationPlayer也已经存在
fn setup_scene(
    _ready: On<SceneInstanceReady>,
    mut commands: Commands,
    animations: Res<Animations>,
    player: Single<(Entity, &mut AnimationPlayer)>,
) {
    // 直接将内部的两个组件的所有权拿出来
    let (entity, mut player) = player.into_inner();
    // 然后我们需要new一个AnimationTransitions
    // 如果你用过css，那么你一定也知道transition有什么用，这里的AnimationTransitions
    // 也是一样的，当动画切换的时候，将会在动画之间插入平滑的过渡
    let mut transitions = AnimationTransitions::new();
	// 立刻开始重复播放第一个动画
    transitions
        .play(&mut player, animations.animations[0], Duration::ZERO)
        .repeat();
	// 这里的commands.entity对应了我们在第2章介绍的EntityComands，还记得吗？
    // 因此我们向scene根实体上，插入了这两个新的组件
    commands
        .entity(entity)
        .insert(AnimationGraphHandle(animations.graph_handle.clone()))
        .insert(transitions);
}

```

这些代码看下来真是让人大汗淋漓。你可能心里会想，怎么那么多没见过的东西？`AnimationNodeIndex`是什么？`AnimationGraph`是什么？`AnimationTransitions`又是什么？`AnimationGraphHandle`又是什么？？

让我们来一个个翻一翻源代码看看吧。`AnimationNodeIndex`其实只是一个别名，其本质上不过是一个含有u32类型的枚举罢了，这个数字标识了一个动画在gltf中的索引。这解释了为什么我们使用`transitions.play`方法时需要他。

```rust
/// The index of either an animation or blend node in the animation graph.
///
/// These indices are the way that [animation players] identify each animation.
///
/// [animation players]: crate::AnimationPlayer
pub type AnimationNodeIndex = NodeIndex<u32>;
```

`AnimationGraph`是一个有向无环图，感兴趣的读者可以查看[文档](https://docs.rs/bevy/latest/bevy/animation/graph/struct.AnimationGraph.html)，他其实相当复杂。描述了多个动画应该如何混合在一起。这是什么意思呢？我们在设计动画时，比如行走和攻击，一般都会设计成为两个单独的动画。如果不能将其混合起来一起播放，那么你行走的时候攻击时角色就会发生诡异的漂移（即脚不动但是角色还在跑，这在很多粗制滥造的游戏中很常见）。

观察其代码，可以发现其由`graph`、`root`、`mask_groups`构成。`AnimationGraphNode`描述了如何对动画进行混合。`HashMap<AnimationTargetId, u64>`则描述了**每块动画骨骼的uuid和掩码组**之间的关系。掩码组是什么意思？

为什么这里是一个`HashMap<AnimationTargetId, u64>`类型呢？其实这是一种典型的位图。u64类型含有64个bit位。因此每个骨骼，最多支持定义 **64 个不同的掩码组**。

举个例子，一个复杂的模型可能有几百根骨骼，但你可能只想给其中的一部分（比如只有右手、或者只有上半身）分配掩码组。`HashMap` 只存储那些被分配了组的骨骼。你在 `mask_groups` 里把所有属于“上半身”的骨骼 ID 都标记为 `1 << 0`（第 0 组）。**节点 A** 播放“奔跑”动画（不带掩码，作用于全身）。**节点 B** 播放“挥剑”或“换弹”动画，但你给这个节点设置一个掩码，指定它**只作用于第 0 组**。角色就可以一边跑（下半身由节点 A 控制），一边做攻击动作（上半身被节点 B 覆盖）。

这方面的混合其实非常复杂，三言两语是搞不定的。如果对动画混合更感兴趣的读者，则需要查阅更详细的资料，这里就不再赘述。

至于`AnimationGraphHandle`，这名字显而易见，由于`AnimationGraph`实际上是一种`asset`，因此当然也有对应的`handle`。

```rust
pub struct AnimationGraph {
    pub graph: Graph<AnimationGraphNode, ()>,
    pub root: NodeIndex,
    pub mask_groups: HashMap<AnimationTargetId, u64>,
}

pub struct AnimationTargetId(pub Uuid);
```

`AnimationTransitions`是一个用于控制动画之间过度的“增强型”`AnimationPlayer`。前面我们说当场景生成时，`AnimationPlayer`会自动添加到场景的任何根动画中。要使用`AnimationTransitions`你还必须把`AnimationGraphHandle`也同时放在一个实体上。仔细一想这也是正常的。因为动画过渡，本身就是利用动画图来实现的。

```rust
// 第一次动画必须使用transitions.play而不能使用AnimationPlayer上的play方法
// 因为AnimationTransitions需要记录动画的顺序，如果直接操作AnimationPlayer开始
// 会导致AnimationTransitions 无法确定动画的切换关系，从而导致过渡效果通常不正确。
let mut transitions = AnimationTransitions::new();
transitions
    .play(&mut player, animations.animations[0], Duration::ZERO)
    .repeat();
// 并且我们把这两个组件都插入到了根上
commands
    .entity(entity)
    .insert(AnimationGraphHandle(animations.graph_handle.clone()))
    .insert(transitions);
```

最后，一切大功告成，只需要在按下不同的按键时切换不同动画即可。

```rust
fn keyboard_control(
    keyboard_input: Res<ButtonInput<KeyCode>>,
    mut animation_players: Query<(&mut AnimationPlayer, &mut AnimationTransitions)>,
    animations: Res<Animations>,
    mut current_animation: Local<usize>,
) {
    // 这里的player是一个AnimationPlayer
    // 而transitions则是我们插入的AnimationTransitions
    for (mut player, mut transitions) in &mut animation_players {
        // 我们可以获取下一个动画的NodeIndex
        let Some((&playing_animation_index, _)) = player.playing_animations().next() else {
            continue;
        };
		// 当按下特定的按键时，可以控制播放器的行为，比如播放暂停
        if keyboard_input.just_pressed(KeyCode::Space) {
            let playing_animation = player.animation_mut(playing_animation_index).unwrap();
            if playing_animation.is_paused() {
                playing_animation.resume();
            } else {
                playing_animation.pause();
            }
        }
        // 还可以控制播放速度
         if keyboard_input.just_pressed(KeyCode::ArrowUp) {
            let playing_animation = player.animation_mut(playing_animation_index).unwrap();
            let speed = playing_animation.speed();
            playing_animation.set_speed(speed * 1.2);
        }
        // 对于动画的切换，我们需要计算下一个索引，并重新使用transitions来切换
        // 必须使用transitions.play()!否则过渡会出问题
        if keyboard_input.just_pressed(KeyCode::Enter) {
            *current_animation = (*current_animation + 1) % animations.animations.len();
            transitions
                .play(
                    &mut player,
                    animations.animations[*current_animation],
                    Duration::from_millis(250),
                )
                .repeat();
        }

       // ..... 等等的类似逻辑
    }
}

```
