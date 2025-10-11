## 2.1 Entity

### 2.1.1 EntityCommands

​	在前面我们生成实体时，还记得我们使用了`Command.spawn()`方法生成了一个没有组件的实体，然后依次往该实体上插入了一些组件，可如何我们想在之后修改实体该怎么办？查看该方法返回的值，可以发现其返回了一个`EntityCommands`类型的值而不是一个`Entity`类型的值，如果要获得真正的`Entity`，我们需要调用`EntityCommands`上的`id`方法。

​	这样设计是因为，单一的`Entity`几乎没有任何用处，真正有用的是与其相关的`EntityCommands`，其本质上是一个有关修改实体命令的队列，利用`EntityCommands`我们可以让我们快速对实体做出一些更改，这些更改大体上分为两类。

1. 与实体上的组件相关的方法
2. 与其他实体相关的方法

​	于此同时，`Commands`上有一个特殊的名为`entity`的方法，该方法接受一个`Entity`类型的参数并返回一个`EntityCommands`类型的值。利用`Commands`，我们可以通过查询系统获得的`Entity`来生成`EntityCommands`。通过这种方式，我们可以实现在游戏运行时**对实体而不是组件**进行修改，例如添加新的组件或者删除旧的组件等。

```rust
fn change_entity(
    mut commands: Commands,
    query: Query<Entity, With<Player>>, // 查找所有有 Player 组件的实体
) {
    for entity in query.iter() {
        let entity_commands = commands.entity(entity);
      	//在这里可以做实体做出修改
      	//..
    }
}
```

### 2.1.2 Relationship

​	之前我们一直侧重于实体与其所含有的组件的关系，但是不同的实体之间也会存在着关系，例如一个玩家可能拥有多个宠物、载具等，当玩家死亡时，这些子实体也应该被重置，这种关系被称为实体之间的`Relationship`。（有意思的是，`Relationship`并不定义在`Entity`上而是在他们的`Component`上）

​	Bevy为我们预先内置了两种关系：`ChildOf`和`children`。前者用于指定当前实体的父实体，后者用于指定当前实体的子实体。通过这样的父子关系，**子实体可以继承父实体的一些组件**，例如可见性或者全局变换。

​	要指定当前实体的父实体，需要我们获取父实体的`Entity`，这可以通过`EntityCommands`上的`id`方法获得。随后，我们使用一个内置的`ChildOf`组件包裹父实体的`Entity`，然后将其添加到子实体身上。当Bevy识别到`ChildOf`组件后，将会自动完成之后的工作，通过其中的`Entity`追踪父实体的生命周期**当父实体销毁时自动销毁子实体**。

```rust
let player = commands.spawn((Player).id();
commands.spawn((Car, ChildOf(player)));
```

​	要指定当前实体的子实体，可以通过通过`EntityCommands`上的`with_children`方法或者`children!`来实现。此外，`EntityCommands`还包含了大量与父子关系相关的方法，通过这些方法还可以动态的删除、替换实体之间的父子关系。

```rust
commands
	.spawn((Player)
  .with_children(|parent| {
			parent.spawn((Car,));
  });
   
//也可以使用宏来完成
commands
	.spawn((Player),
  children![
      (Car,),
      (Car,),
  ]);
```

​	除了父子关系外，实体之间还可能有着其他各种各样的关系，因此Bevy还提供了更高级的 API抽象，让我们能够自定义实体之间的关系并决定如何处理这种关系。要自定义关系，我们需要定义关系的`relationship`与`relationship_target`，前者作为关系的“源”，后者作为关系的“目标”。这类似于数据库关系中的One-to-Many关系，前者即是关系中的one，后者是关系中的many。

```rust
// 定义一个关系的“源”，一个“源”只能引用一个实体
#[derive(Component, Debug)]
#[relationship(relationship_target = TargetedBy)]
struct Targeting(Entity);

//定义一个关系的“目标”，由于一个目标会有多个相关联的实体，因此这里是Vec<Entity>
//在这里我们启用linked_spawn后，能够让Bevy在target销毁时自动清除其内的关联实体
#[derive(Component, Debug)]
#[relationship_target(relationship = Targeting，linked_spawn)]
struct TargetedBy(Vec<Entity>);
```

​	有了这些，我们便可以通过将其作为组件来使用

```rust
fn spawn_player(mut commands: Commands) {
  let player = commands.spawn((Player, Name::new("player_one"))).id();
	//使用Targeting代表关系中的“源”
  commands.spawn((Car, Targeting(player), Name::new("Lamborghini")));
  commands.spawn((Pet, Targeting(player), Name::new("Black")));
}

commands.spawn((
  Player,
  Name::new("player_one"),
  related!(TargetedBy[
    // 使用related!宏和TargetedBy直接从关系目标实体上定义关系
    (Car, Name::new("Lamborghini")),
    (Pet, Name::new("Black")),
  ]),
));
```

​	回想一下数据库中的基础知识，现在我们能够解决One-to-Many的关系情况了，如何解决Many-to-Many关系的呢？在数据库中，这往往通过新增一个关系表来实现，类比这种方法，在Bevy中我们也可以通过新增一个关系实体。我们可以利用最经典的学生与课程的例子来讲解。

​	我们将Many-to-Many分解为两个One-to-Many关系，并将其中的两个One组件插入到一个关系实体上，这样我们便可以借助连接实体“顺藤摸瓜”得到对应的学生和课程。

```rust
// 实体 Student
#[derive(Component)] struct Student;
// 实体 Course
#[derive(Component)] struct Course;

// 学生实体上的 Relationship
#[derive(Component)]
#[relationship(relationship_target = StudentEnrollments)]
struct JunctionToStudent(Entity);

// 课程实体上的 Relationship
#[derive(Component)]
#[relationship(relationship_target = CourseEnrollments)]
struct JunctionToCourse(Entity);

// 学生实体上的 RelationshipTarget
#[derive(Component)]
#[relationship_target(relationship = JunctionToStudent)]
struct StudentEnrollments(Vec<Entity>);

// 课程实体上的 RelationshipTarget
#[derive(Component)]
#[relationship_target(relationship = JunctionToCourse)]
struct CourseEnrollments(Vec<Entity>);


let student_a = commands.spawn(Student).id();
let course_math = commands.spawn(Course).id();

// 创建连接实体
commands.spawn((
    JunctionToStudent(student_a),
    JunctionToCourse(course_math),
));
```

## 2.2 Component

### 2.2.1 Archetype

​	实体与组件的关系可以类比成数据库中表，其中每一行代表了`world`中的某个实体与其相关的组件，每一列代表了其中的组件。然而，这只是我们一厢情愿的类比，这些实体和组件在Bevy中真正的存储方式要比这复杂的多。

​	Bevy将组件默认存储在`Table`中，从概念上讲，`Table`只是一种用于储存数据的数据结构，类似于一个`HashMap<ComponentId, Column>`，其中每个`Column`是一个`Vec<T:Component>`，这意味着其可以方便的查找，但是不方便组件的插入与删除（想象一下，当你向一个Vec中间插入一个新元素的时候，你需要把新位置后的元素全都向后平移一位以腾出位置）。因此，Bevy还提供了一种`SparseSet`形式的存储数据结构，使用这种稀疏数据结构，可以在需要频繁的插入与删除时提高性能。

```rust
#[derive(Component)]
//如果一个组件可能被频繁插入或者删除，可以标记为稀疏集来优化性能
#[component(storage = "SparseSet")]
struct SomeComponent;
```

​	想象一下在这样的数据结构中我们如何查找一个满足要求的行？必须首先从通过组件的`ComponentId`获取这一列，然后获取该列中实体的行。当我们需要查找的实体要满足很多条件时，重复进行这样的查找和修改是十分低效的，这是一个不能并行的操作，因为我们不能确定另外一个列位置上是否拥有同样的属性（为什么？）。

​	为了解决这个问题，Bevy中引入了`Archetype`。从技术上讲，`Archetype`是固定组件的组合，这意味着一个`Archetype`内的实体，其**拥有的组件种类是相同的**，这使得的Bevy能够通过矢量化操作提高查找或者修改的效率从而提高性能。Bevy在`Archetype`存储在了一个`Table`中的引用，这意味着多个`Archetype`可以共享一张表，但是每个`Archetype`都只指向一张`Table`。

​	作为一个例子，考虑下面的这张`Table`，当我们查询`Player`时如果没有`Archetype`，那么我们必须遍历每个实体才能找到我们最终的结果。

| Entity | Player | Monster | Health | Attack |
| :----: | :----: | :-----: | :----: | :----: |
|   1    |   ✓    |    -    |  100   |   10   |
|   2    |   -    |    ✓    |   50   |   15   |
|   3    |   -    |    ✓    |   75   |   20   |
|   4    |   ✓    |    -    |   80   |   15   |

​	现在，我们可以将其分为两个`Archetype`并引用上面表中的数据。当我们需要找到拥有`Player`的实体时，我们可以根据原型直接排除第一个`Archetype`上的所有实体。完美！现在我们的查询系统可以通过`Archetype`知道那些实体拥有同样的结构以此来对操作进行并行加速了。

| Entity | Monster | Health | Attack |
| :----: | :-----: | :----: | :----: |
|   1    |    ✓    |   50   |   15   |
|   2    |    ✓    |   75   |   20   |

| Entity | Player | Health | Attack |
| :----: | :----: | :----: | :----: |
|   1    |   ✓    |  100   |   10   |
|   2    |   ✓    |   80   |   15   |

​	尽管这些工作是引擎的幕后工作，但是了解Bevy是如何组织我们的数据是非常重要的，这使得我们能更好的够优化自己的数据组织方式来帮助Bevy更快的运行我们的程序。

### 2.2.2 Bundle

​	很多初次学习Bevy的人往往都弄不清楚`Bundle`和`component`的关系，从字面意思上来看，`Bundle`的含义是“一堆，一批”，从实际功用上来看，`Bundle`是一个容器，容纳了一组`component`。

​	还记得我们是如何往实体上插入属性的吗？通过`spawn`方法，我们传入了一个元组，如果你细心，你能够发现`spawn`方法接受的参数类型是一个实现了`Bundle`特型的泛型。由于Bevy为元组类型实现了该特型，因此在这里你不要自己声明`Bundle`。我们也可以自己声明需要的`Bundle`，只需要使用`Bundle`指令即可。

​	利用`Bundle`，最直观的便捷就是我们可以快速插入或者删除一组组件。不过，**不能在查询中使用`Bundle`**，这是因为查询系统需要访问其中的各个组件类型才能过滤实体。

```rust
commands.spawn((Player,Health::new(100),Attack::new(10)))

//实际上这些代码等同于下面这些
#[derive(Bundle)]
struct PlayerBundle {
  player: Player,
  health: Health,
  attack: Attack
}
//一次性插入多个组件
let player = commands
  .spawn_empty()
  .insert(PlayerBundle {
    player: Player,
    health: Health::new(100),
    attack: Attack::new(10)
  })
	.id();

//一次性删除这些组件
commands.entity(player).remove::<PlayerBundle>();
```

​	不过，既然Rust能够正确推断出类型，为什么我们还要自己手动实现需要的`Bundle`结构体呢？答案是`Bundle`能够帮我们更好的管理组件的结构帮助我们管理代码和实体。

​	当一个`Bundle`的字段里含有另一个`Bundle`时会发生什么（这是常见的，因为我们这可以复用我们的代码并解耦组件之间的以来）？我们知道所有的组件在Bevy中都是扁平化的，一个组件不可能包含另一个组件。因此Bevy会自动帮我们将其展开。但需要注意的是，**不能包含一个`Bundle`多次，否则Bevy将会崩溃**。

```rust
#[derive(Bundle)]
struct BiologyBundle {
  health: Health,
  attack: Attack
}

#[derive(Bundle)]
struct PlayerBundle {
  player: Player,
  health_and_attack: BiologyBundle,
}

//当我们拥有以上定义时，Bevy会自动帮我们扁平化PlayerBundle，生成下面的结构
//struct PlayerBundle {
//  player: Player,
//  health: Health,
//  attack: Attack
//}
```

### 2.2.3 require

​	`require`属性的组件会使插入一个组件时如果实体身上没有需要的组件时，自动插入其他的组件（必须实现`Default`或在指令中指定值）。

```rust
#[derive(Component)]
//下面这些方式都可以定义必须组件
#[require(Health, Attack)]
#[require(Health = Health{100},Health = Attack{100}]
struct Player;

//当我们插入Player时，Health和Attack也会被一并插入
let player = commands.spawn(Player).id();

//我们可以获得其身上自动插入的属性
commands.entity(player).get::<Health>().unwrap();
```

​	不过，这种情况也会导致一些类似于面向对象中的“多重继承”的问题，即一个组件通过多个`require`链产生了同一组件多次。一般来说，我们要避免这种情况的发生，不过当无法避免时，Bevy将遵循以下的初始化顺序。

1. 如果`#[require()]`中存在明显的构造函数，则优先选择该构造函数。
2. 否则，对`require`树执行深度优先搜索并选择找到的第一个。

​	以上的方式通过在编译时生成必须组件，当我们在运行时需要指定必须组件时，可以调用`World`上的`register_required_components`或`register_required_components_with`方法，具体的使用方式可以查询Bevy文档即可，这里不再赘述。

## 2.3 System



## 2.4 Query



## 2.6 Schedule



## 2.5 World与Command

