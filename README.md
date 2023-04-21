# 游戏开发中的ECS 架构



## ESC简介

ECS, 即 Entity-Component-System (实体-组件-系统) 的缩写，其模式遵循组合优于继承的原则。游戏里每个基本单位都是一个实体，每个实体又由一个或者多个组件组件构成，每个组件仅仅包含代表其特性的数据（即在组件中没有任何方法），例如：移动相关组件 MoveComponent 包含速度，位置，朝向等属性。一旦一个实体拥有了 MoveComponent 组件，便可以认为它拥有了移动的能力， 系统便是来处理拥有一个或者多个相同组件的实体集合的工具，其只拥有行为（即在系统中没有任何数据），在这个例子中，处理移动的系统仅仅关心拥有移动能力的实体，他会遍历所有拥有 MoveComponent 组件的实体，并根据相关的数据（速度，位置，朝向等），更新实体的位置。

实体与组件是一个 **一对多** 的关系，实体拥有怎么样的能力，完全是取决于其拥有哪些组件，通过动态添加或者删除组件，可以在（游戏）运行时，改变该 实体 的行为。



## ECS基本结构

一个使用ECS架构开发的游戏基本结构如下图所示：

![](.\img\1.jpg)

先有一个 Wold，它是系统和实体的集合，而实体就是一个ID，这个ID对应了组件的集合，组件用来存储游戏状态并且没有任何行为，系统拥有处理实体的行为但是没有状态。



## 详解ECS中的实体，组件和系统

### 实体

实体只是一个概念上的定义，指的是存在你游戏世界中的一个独特物体，是一系列组件的集合。为了方便区分不同的实体，在代码层面上一般用一个ID来表示，所有组成这个实体的组件将会被这个ID标记，从而明确哪些组件属于这个实体。由于其是一系列组件的集合，因此完全可以在运行时动态的为实体增加一个新的组件或者是将组件从实体中移除。比如：玩家实体因为某些原因，丧失了移动能力，主需要简单的将移动组件从该实体上移除，便可以达到无法移动的目的。

#### 样例

- Player(Position, Sprite, Velocity, Health)
- Enemy(Position, Sprite, Velocity, Health, AI)
- Tree(Position, Sprite)

*注：括号前为实体名，括号内为该实体拥有的组件*



### 组件

一个组件是一堆数据的集合，可以使用C++里的结构体来进行实现。它没有方法，即不存在任何行为，只用来存储状态。一个经典的实现是：每一个组件都继承（或者实现）同一个基类（或者接口），通过这样的方法，我们能够非常方便的在运行时动态添加，识别，移除组件。每一个组件的意义在于描述实体的某一个特性，例如：PositionComponent （位置组件)，其拥有x，y两个数据，用来描述实体的位置信息，拥有 PositionComponent 的实体便可以说在游戏世界中拥有了一席之地，当组件们单独 存在的时候，实际上是没有什么意义的，但是当多个组件通过系统的方式组织在一起，才能发挥真正的作用。同时，我们还可以用空组件（不包含任何数据的组件）对实体进行标记，从而在运行时动态的识别它。如：EnmeyComponent 这个组件可以不包含任何数据，拥有该组件的实体被标记为敌人。

*一个变换组件例子*

```c++
struct TransformComponent
	{
		glm::mat4 Transform{1.0f};

		TransformComponent() = default;
		TransformComponent(const TransformComponent&) = default;

		TransformComponent(const glm::mat4& transform) : Transform(transform) {}

		operator glm::mat4& () { return Transform; }
		operator const glm::mat4& () const { return Transform; }
	};
```

*该变换组件，里面有一个变换矩阵属性，通过改变该属性，实现各种变换（缩放，位移，旋转），拥有该组件的实体，都能通过更新该属性矩阵来实现变换功能*

根据实际开发需求，这里还会存在一种特殊的组件，名为 **Singleton Component （单例组件）**，顾名思义，单例组件在一个上下文中有且只有一个。具体在什么情况下使用在系统一节中会提到。

**样例**：

- PositionComponent(x, y)
- VelocityComponent(X, y)
- HealthComponent(value)
- PlayerComponent()
- EnemyComponent()

*注：括号前为组件名，括号内为该组件拥有的数据*



### 系统

理解了实体和组件便会发现，至此还未曾提到过游戏逻辑相关的话题。系统便是ECS架构中用来处理游戏逻辑的部分。何为系统，一个系统就是对拥有一个或多个相同组件的实体集合进行操作的工具，它只有行为，没有状态，即不应该存放任何数据。举个例子，游戏中玩家要操作对应的角色进行移动，由上面两部分可知，角色是一个实体，其拥有位置和速度组件，那么怎么根据实体拥有的速度去刷新其位置呢，`MoveSystem`（移动系统）登场，它可以得到所有拥有位置和速度组件的实体集合，遍历这个集合，根据每一个实体拥有的速度值和物理引擎去计算该实体应该所处的位置，并刷新该实体位置组件的值，至此，完成了玩家操控的角色移动了。

注意，这里强调了移动系统可以得到**所有**拥有位置和速度组件的实体集合，因为一个实体同时拥有位置和速度组件，我们便认为该实体拥有移动的能力，因此移动系统可以去刷新每一个符合要求的实体的位置。这样做的好处在于，当我们玩家操控的角色因为某种原因不能移动时，我们只需要将速度组件从该实体中移除，移动系统就得不到角色的引用了，同样的，如果我们希望游戏场景中的某一个物件动起来，只需要为其添加一个速度组件就万事大吉。

一个系统关心实体拥有哪些组件是由我们决定的，通过一些手段，我们可以在系统中很快地得到对应实体集合。

上文提到的 **Singleton Component （单例组件）** ，明白了系统的概念更容易说明，还是玩家操作角色的例子，该实体速度组件的值从何而来，一般情况下是根据玩家的操作输入去赋予对应的数值。这里就涉及到一个新组件`InputComponent`（输入组件）和一个新系统`ChangePlayerVelocitySystem`（改变玩家速度系统），改变玩家速度系统会根据输入组件的值去改变玩家速度，假设还有一个系统`FireSystem`（开火系统），它会根据玩家是否输入开火键进行开火操作，那么就有 2 个系统同时依赖输入组件，真实游戏情况可能比这还要复杂，有无数个系统都要依赖于输入组件，同时拥有输入组件的实体在游戏中仅仅需要有一个，每帧去刷新它的值就可以了，这时很容易让人想到单例模式（便捷地访问、只有一个引用），同样的，单例组件也是指整个游戏世界中有且只有一个实体拥有该组件，并且希望各系统能够便捷的访问到它，经过一些处理，在任何系统中都能通过类似`world->GetSingletonInput()`的方法来获得该组件引用。

系统这里比较麻烦，还存在一个常见问题：由于代码逻辑分布于各个系统中，各个系统之间为了解耦又不能互相访问，那么如果有多个系统希望运行同样的逻辑，该如何解决，总不能把代码复制 N 份，放到各个系统之中。**UtilityFunction**（实用函数） 便是用来解决这一问题的，它将被多个系统调用的方法单独提取出来，放到统一的地方，各个系统通过 **UtilityFunction** 调用想执行的方法，同系统一样， **UtilityFunction** 中不能存放状态，它应该是拥有各个方法的纯净集合。

**样例**：

- MoveSystem(Position, Velocity)
- RenderSystem(Position, Sprite)

*注：括号前为系统名，括号内为该系统关心的组件集合*

---



## ECS优劣

ECS除了比较熟知的性能优势外，还有一个更重要也更实用的点是方便做状态快照。

由于Component是只包含数据的，而System是只包含逻辑的，这种数据和逻辑必须独立实现的结构使得开发者必须以“状态”的概念来设计组件。而一旦满足了这样的设计，就使得游戏运行只需要基于当前Component的成员变量就能推断出后续的变化。也就是说，独立管理所有数据的Component天生就是一个完美的状态的“快照”。

而这种天生支持独立状态快照的结构，对于做联机同步、存档还原、多线程等，都是非常便利的。

而ECS的缺点，就在于它对开发规范化的要求有点高，而绝大多数开发者对于ECS的优势只有一个模糊的概念。此消彼长，使得大多数开发者在开发过程中，并不能坚定且正确低贯彻ECS的开发原则。

所谓对开发规范化的要求有点高，是因为ECS不同于OO或者COM这种概念上的“架构”，ECS并没有从编译级别上去强制要求开发者遵循自己的原则。这就使得在开发过程中，很容易出现一些“偷懒”“绕道”的现象。而随着这些“偷懒”“绕路”慢慢累积，最后会发现根本无法发挥ECS的上述那些优势。

而之所以说绝大多数开发者只是对ECS的优点有一个模糊的概念，是因为ECS的这些优势只有在达到一定规模体量的情况下才能体现。然而对于很多游戏项目来说，可能性能并不是一个瓶颈，或者性能瓶颈并不在大规模数据逻辑的overhead上。那这时候使用ECS，开发者反而会很容易陷入一种迷茫：我为啥要用ECS？

简而言之，ECS的的问题在于，它这些种种特性，使得正常人第一次决定使用这个架构的时候，过程和结局往往都不是那么幸福，而这就会导致大多数人不是浅尝辄止，就是望而生畏。所以如果要使用ECS开发并发挥其优势，对于开发团队的要求还是比较高的。