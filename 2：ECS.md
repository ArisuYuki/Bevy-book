## 1.1 Entity

### 1.1.1 修改实体

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

### 1.1.2 关系

​	之前我们一直侧重于实体与其所含有的组件的关系，但是不同的实体之间也会存在着关系，例如一个玩家可能拥有多个宠物、载具等，当玩家死亡时，这些子实体也应该被重置，这种关系被称为实体之间的`Relationship`。

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





## 1.2 Component

### 1.2.1 组件在Bevy中的组织方式

​	实体与组件的关系可以类比成数据库中表，其中每一行代表了`world`中的某个实体与其相关的组件。



## 1.3 System



## 1.4 Query



## 1.6 Schedule



## 1.5 Command

