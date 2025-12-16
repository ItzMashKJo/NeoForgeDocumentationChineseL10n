这一页旨在让物品被玩家右击这一实在复杂而令人困惑的过程更容易被理解，同时阐明于何处使用什么结果（译注：结果对象）以及为什么。

## 我右击时发生了什么？
当你在世界的任何地方右击时，一些事情发生了，它们取决于你目前看向什么东西以及你手上的`ItemStack`是什么。一些返回两种结果类型（见下文）的其中之一的方法被调用了。大部分方法会取消管线，如果一个明确的成功或明确的失败结果被返回了。为了可阅读性，从现在开始，这个“明确的成功或明确的失败结果”会被称为“决定性的结果”。

* `InputEvent.InteractionKeyMappingTriggered`由鼠标右键和主手触发。如果事件被取消，管线终止。
* 数个条件被检查，例如你不处于观察者模式，或者你主手上的`ItemStack`需要的所有特性标志都被开启了。如果至少有一个检查失败了，管线终止。
* 取决于你在看向什么，不同的事情会发生：
	* 如果你在看向一个处于你可及范围内的实体，且不在世界边界外：
		* `PlayerInteractEvent.EntityInteractSpecific`被触发。如果事件被取消，管线终止。
		* `Entity#interactAt`会在**你看向的实体**身上被调用。如果它返回了一个决定性的结果，管线终止。
			* 如果你想要给你自己的实体添加行为，覆写该方法；如果你想要给原版的实体添加行为，用事件。
		* 如果该实体打开了一个界面（例如一个村民交易GUI或一个箱子矿车GUI），管线终止。
		* `PlayerInteractEvent.EntityInteract`被触发。如果事件被取消，管线终止。
		* `Entity#interact`会在**你看向的实体**身上被调用。如果它返回了一个决定性的结果，管线终止。
			* 如果你想要给你自己的实体添加行为，覆写该方法；如果你想要给原版的实体添加行为，用事件。
			* 对于`Mob`，当你的主手上的`ItemStack`是一个刷怪蛋时，`Entity#interact`的覆写处理了拴绳、生成幼体的逻辑，接着延迟生物特定的逻辑处理到`Mob#mobInteract`。`Entity#interact`的结果的规则在这里也适用。
		* 如果你看向的实体是一个`LivingEntity`，`Item#interactLivingEntity`会在你的主手上的`ItemStack`上被调用。如果它返回了一个决定性的结果，管线终止。
	* 如果你在看向一个处于你可及范围内的方块，且不在世界边界外：
		* `PlayerInteractEvent.RightClickBlock`被触发。如果事件被取消，管线终止。在该事件中，你也可以特定地只拒绝方块或物品的使用。
		* `IItemExtension#onItemUseFirst`被调用。如果它返回了一个决定性的结果，管线终止。
		* 如果玩家没有在潜行，且事件没有拒绝方块的使用，`UseItemOnBlockEvent`被触发。如果事件被取消，取消结果会被采用；否则，`Block#useItemOn`被调用。如果它返回了一个决定性的结果，管线终止。
		* 如果`ItemInteractionResult`是`PASS_TO_DEFAULT_BLOCK_INTERACTION`且执行的手是主手，接着`Block#useWithoutItem`被调用。如果它返回了一个决定性的结果，管线终止。
		* 如果事件没有拒绝物品的使用，`Item#useOn`被调用。如果它返回了一个决定性的结果，管线终止。
* `Item#use`被调用。如果它返回了一个决定性的结果，管线终止。
* 以上的过程运行第二次，这一次不是主手，而是副手。

## 结果类型
有三种不同类型的结果：`InteractionResult`、`ItemInteractionResult`和`InteractionResultHolder<T>`。大部分情况下使用`InteractionResult`，只有`Item#use`使用`InteractionResultHolder<ItemStack>`，只有`BlockBehaviour#useItemOn`和`CauldronInteraction#interact`使用`ItemInteractionResult`。

`InteractionResult`是一个有五个值的枚举：`SUCCESS`、`CONSUME`、`CONSUME_PARTIAL`、`PASS`和`FAIL`。另外，还有`InteractionResult#sidedSuccess`可用，它在服务端返回`SUCCESS`，在客户端返回`CONSUME`。

`InteractionResultHolder<T>`是一个`InteractionResult`的包装类，添加了额外的T类型的上下文。T可以是任何类型，但99.99%的情况下，它是`ItemStack`。`InteractionResultHolder<T>`为枚举值提供了包装方法（`#success`、`#consume`、`#pass`和`#fail`），以及`#sidedSuccess`，它在服务端调用`#success`，在客户端调用`#consume`。

`ItemInteractionResult`与`InteractionResult`类似，专用于一个物品使用于一个方块的情况。它是一个有六个值的枚举：`SUCCESS`、`CONSUME`、`CONSUME_PARTIAL`、`PASS_TO_DEFAULT_BLOCK_INTERACTION`、`SKIP_DEFAULT_BLOCK_INTERACTION`和`FAIL`。每个`ItemInteractionResult`可以通过`#result`映射为一个`InteractionResult`；`PASS_TO_DEFAULT_BLOCK_INTERACTION`和`SKIP_DEFAULT_BLOCK_INTERACTION`都代表`InteractionResult#PASS`。相似的是，`ItemInteractionResult`同样存在`#sidedSuccess`。

通常，这些不同的值代表如下含义：

* `InteractionResult#sidedSuccess`（或视情况使用`InteractionResultHolder#sidedSuccess`/`ItemInteractionResult#sidedSuccess`）应该被使用在操作应当被视为成功，且你想让手臂摆动的情况。管线会终止。
* `InteractionResult#SUCCESS`（或视情况使用`InteractionResultHolder#success`/`ItemInteractionResult#SUCCESS`）应该被使用在操作应当被视为成功，且你只想在一个逻辑端上让手臂摆动的情况。只有当你不知出于何种原因，想要返回一个不同的值于另外一个逻辑端时，才使用这个。管线会终止。
* `InteractionResult#CONSUME`（或视情况使用`InteractionResultHolder#consume`/`ItemInteractionResult#CONSUME`）应该被使用在操作应当被视为成功，但你不想让手臂摆动的情况。管线会终止。
* `InteractioinResult#CONSUME_PARTIAL`在大多数情况下和`InteractionResult#CONSUME`等同，唯一的区别在于它在`Item#useOn`中的用途。
	* `ItemInteractionResult#CONSUME_PARTIAL`在`BlockBehaviour#useItemOn`中的用途是类似的情况。
* `InteractionResult#FAIL`（或视情况使用`InteractionResultHolder#fail`/`ItemInteractionResult#FAIL`）应该被使用在物品的功能应当被视为失败了，且没有进一步的交互应当被执行的情况。管线会终止。这可以被使用在任何地方，但在`Item#useOn`和`Item#use`之外的地方应当小心使用。在很多情况下，用`InteractionResult#PASS`更说得通。
* `InteractionResult#PASS`（或视情况使用`InteractionResultHolder#pass`）应该被使用在操作应当被视为既不成功也不失败的情况。管线会继续。这是默认的行为（除非另有说明）。
	* `ItemInteractionResult#PASS_TO_DEFAULT_BLOCK_INTERACTION`允许`BlockBehaviour#useWithoutItem`在主手上被调用，同时`#SKIP_DEFAULT_BLOCK_INTERACTION`完全阻止该方法被执行。`#PASS_TO_DEFAULT_BLOCK_INTERACTION`是默认的行为（除非另有说明）。


## IItemExtension#onItemUseFirst
`InteractionResult#sidedSuccess`和`InteractionResult#CONSUME`在这里没有效果。只有`InteractionResult#SUCCESS`、`InteractionResult#FAIL`或`InteractionResult#PASS`应该被用在这里。

## Item#useOn
如果你希望操作应当被视为成功，但你不想让手臂摆动或奖励一个`ITEM_USED`点数，使用`InteractionResult#CONSUME_PARTIAL`。

## Item#use
这是返回类型为`InteractionResultHolder<ItemStack>`的唯一实例。返回的`InteractionResultHolder<ItemStack>`中的`ItemStack`替代了使用操作时发起时的`ItemStack`，如果它发生了变化的话。

`Item#use`的默认实现是，如果物品可被食用，且玩家可以食用该物品（因为玩家饿了，或该物品始终可被食用），返回`InteractionResultHolder#consume`；如果物品可被食用但玩家不能食用，返回`InteractionResultHolder#fail`；如果物品不可被食用，返回`InteractionResultHolder#pass`。

如果要让主手阻止副手行为的运行，返回`InteractionResultHolder#fail`。如果你想要副手行为运行（这通常是你想要的），返回`InteractionResultHolder#pass`以替代。
