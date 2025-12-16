工具是主要作用为破坏方块的物品。许多Mod添加了新的工具集（例如铜工具）或新的工具类型（例如锤子）。

## 自定义工具集
一个工具集一般由五个物品组成：一把镐子，一把斧头，一把铲子，一把锄头和一把剑（剑传统意义上并非工具，但为了一致性，也被包括在这里）。所有这些物品有它们对应的类：分别是`PickaxeItem`、`AxeItem`、`ShovelItem`、`HoeItem`和`SwordItem`。工具类的继承关系看起来就像下面这样：

```
Item
- TieredItem
    - DiggerItem
        - AxeItem
        - HoeItem
        - PickaxeItem
        - ShovelItem
    - SwordItem
```

`TieredItem`是一个包含了给带有特定`Tier`（继续读下去）的物品的帮助方法的类。`DiggerItem`包含了给被设计为破坏方块的物品的帮助方法。注意，其他通常被视为工具的物品，如剪刀，并不被包含在这个继承树中。相反，它们直接继承`Item`，自己持有破坏逻辑。

要创建一个标准的工具集，你必须首先定义一个`Tier`。对于用作参考的值，见Minecraft的`Tiers`枚举。这个示例使用铜工具，这里你可以使用你自己的材料，按需调整值。

```Java
//我们将铜放在石头和铁之间的某个位置。
public static final Tier COPPER_TIER = new SimpleTier(
        //这个标签决定了这个工具不能破坏什么方块。更多信息见下。
        MyBlockTags.INCORRECT_FOR_COPPER_TOOL,
        //决定该等级的耐久。
        //石头是131，铁是250。
        200,
        //决定该等级的挖掘速度。剑不使用该数值。
        //石头使用4，铁使用6。
        5f,
        //决定额外攻击伤害。不同的工具不同地使用该数值。例如，剑造成(getAttackDamageBonus() + 4)点伤害。
        //石头使用1，铁使用2，对于剑，分别对应到5点和6点攻击伤害；我们的剑现在造成5.5点伤害。
        1.5f,
        //决定该等级的附魔能力。这代表了该工具上的魔咒会有多好。
        //金使用22，我们把铜放在稍微低一点的位置。
        20,
        //决定该等级的修复原料。为了惰性初始化，使用一个supplier。
        () -> Ingredient.of(Tags.Items.INGOTS_COPPER)
);
```

现在我们有了我们的`Tier`，我们可以使用它来注册工具。所有工具的构造器有相同的四个参数。

```Java
//ITEMS是一个DeferredRegister<Item>
public static final Supplier<SwordItem> COPPER_SWORD = ITEMS.register("copper_sword", () -> new SwordItem(
        //所用的等级。
        COPPER_TIER,
        //物品属性。我们不需要在这里设定耐久值，因为TieredItem类为我们处理了它。
        new Item.Properties().attributes(
            //对于每个DiggerItem，`createAttributes`方法既存在于对应的类中，也存在于其子类中
            SwordItem.createAttributes(
                //所用的等级。
                COPPER_TIER,
                //特定于类型的额外攻击伤害。剑用3，铲子用1.5，镐子用1，斧头和锄头的这个值是变化的。
                3,
                //特定于类型的攻击速度修饰符。玩家有一个默认的4点攻击速度，所以为了达到期望的值1.6f，我们使用-2.4f。剑用-2.4f，铲子用-3.0f，镐子用-2.8f，斧头和锄头的这个值是变化的。
                -2.4f,
            )
        )
));
public static final Supplier<AxeItem> COPPER_AXE = ITEMS.register("copper_axe", () -> new AxeItem(...));
public static final Supplier<PickaxeItem> COPPER_PICKAXE = ITEMS.register("copper_pickaxe", () -> new PickaxeItem(...));
public static final Supplier<ShovelItem> COPPER_SHOVEL = ITEMS.register("copper_shovel", () -> new ShovelItem(...));
public static final Supplier<HoeItem> COPPER_HOE = ITEMS.register("copper_hoe", () -> new HoeItem(...));
```

### 标签
当创建一个`Tier`时，它被分配了一个方块标签，包含了如果被该工具破坏不会掉落任何东西的方块。例如，`minecraft:incorrect_for_stone_tool`标签包含了像钻石矿石之类的方块，`minecraft:incorrect_for_iron_tool`标签包含了像黑曜石和远古残骸之类的方块。为了让指派方块到它们的不正确的采掘等级更加容易，一个需要这个工具来采掘的方块的标签同样存在。例如，`minecraft:needs_iron_tool`标签包含了像钻石矿石之类的方块，`minecraft:needs_diamond_tool`标签包含了像黑曜石和远古残骸之类的方块。

你可以为你的工具重用这些“不正确”标签之一，如果这对你来说不要紧。例如，如果我们希望我们的铜工具只是耐久更多的石制工具，我们传入`BlockTags#INCORRECT_FOR_STONE_TOOL`。

或者，我们可以创建我们自己的标签，像这样：

```Java
//这个标签将允许我们将这些方块添加到无法挖掘它们的不正确标签中
public static final TagKey<Block> NEEDS_COPPER_TOOL = TagKey.create(BuiltInRegistries.BLOCK.key(), ResourceLocation.fromNamespaceAndPath(MOD_ID, "needs_copper_tool"));

//这个标签会被传入我们的tier
public static final TagKey<Block> INCORRECT_FOR_COPPER_TOOL = TagKey.create(BuiltInRegistries.BLOCK.key(), ResourceLocation.fromNamespaceAndPath(MOD_ID, "incorrect_for_cooper_tool"));
```

接着，我们为我们的标签填充数据。例如，让我们让铜（工具）能够采掘金矿石、金块和红石矿石，但不能采掘钻石（矿石）和绿宝石（矿石）。（红石块已经能被石制工具采集。）标签文件位于`src/main/resources/data/mod_id/tags/block/needs_copper_tool.json` （其中`mod_id`是你的Mod ID。

```JSON
{
    "values": [
        "minecraft:gold_block",
        "minecraft:raw_gold_block",
        "minecraft:gold_ore",
        "minecraft:deepslate_gold_ore",
        "minecraft:redstone_ore",
        "minecraft:deepslate_redstone_ore"
    ]
}
```

接着，对于我们的传入tier的标签，我们可以设定一个负面约束，用以排除那些被我们的铜工具标签包含，但对石制工具而言却属无效挖掘的方块。标签文件位于`src/main/resources/data/mod_id/tags/block/incorrect_for_copper_tool.json`：

```JSON
{
    "values": [
        "#minecraft:incorrect_for_stone_tool"
    ],
    "remove": [
        "#mod_id:needs_copper_tool"
    ]
}
```

最终，我们可以将我们的标签传递进我们的tier创建，正如上面所看到的那样。

如果你想要检查一个工具是否能让一个方块状态掉落其方块（物品），调用`Tool#isCorrectForDrops`。`Tool`对象可以由以`DataComponents#TOOL`调用`ItemStack#get`得到。

## 自定义工具
自定义的工具可以通过以`Item.Properties#component`添加一个`Tool`（通过`DataComponents#TOOL`）到你的物品的默认组件列表来创建。`DiggerItem`是一个接受一个`Tier`来构造`Tool`的实现，正如上文所述。`DiggerItem`也提供了一个方便的名为`#createAttributes`的方法来提供给`Item.Properties#attributes`给你的工具，例如被修饰的攻击伤害和攻击速度。

一个`Tool`包含一个`Tool.Rule`的列表，手持工具时默认的挖掘速度（默认为1），以及工具采掘一个方块时应当受到的损害值（默认为1）。一个`Tool.Rule`包含三条信息：一个应当应用该规则的方块的`HolderSet`，一个可选的采掘集合中的方块的速度，以及一个可选的布尔值，用于决定这些方块被该工具采掘时是否可以掉落。如果可选值未被设定，接着其他规则会被检查。如果所有规则都检查失败，则默认行为是（采取）默认的挖掘速度，且方块不能掉落。

> 注：一个`HolderSet`可以通过`Registry#getOrCreateTag`由一个`TagKey`创建。

创建一个多用工具物品（换句话说，一个将两个或多个工具属性合为一体的物品，例如一个物品同时是一个斧头和一个镐子）或任何类似于工具的物品不需要继承任何的已有`TieredItem`类。它可以由使用以下这些部分的组合实现：

* 通过由`Item.Properties#component`设定`DataComponents#TOOL`，添加一个有你自己的规则的`Tool`。
* 通过`Item.Properties#attributes`来添加属性（例如攻击伤害、攻击速度）给物品。
* 覆写`IItemExtension#canPerformAction`来决定物品可以表现出什么`ItemAbility`。
* 调用`IBlockExtension#getToolModifiedState`，如果你希望你的物品基于`ItemAbility`右键时修改方块状态。
* 将你的物品添加到一些`minecraft:enchantable/*`标签中，这样你的物品可以应用某些魔咒。

## ItemAbility
`ItemAbility`是对物品可以做什么和不可以做什么的抽象。这都包括了左击和右击行为。NeoForge在`ItemAbilities`类中提供了默认的`ItemAbility`：

* 挖掘能力。这些存在于上文提到的所有四种`DiggerItem`类型，以及剑和剪刀的挖掘。
* 斧头右击能力，针对去皮（原木）、除锈（氧化的铜）和脱蜡（涂蜡的铜）。
* 剪刀的能力，针对采收（蜜脾）、雕刻（南瓜）和拆除引信（绊线钩）。
* 铲子压扁（草径）的能力，剑横扫的能力，锄头耕作的能力，盾牌格挡的能力，钓鱼竿投射（鱼钩）的能力。

要创建你自己的`ItemAbility`，使用`ItemAbility#get` - 它会按需创建一个新的`ItemAbility`。接着，在一个自定义工具类型中，按需覆写`IItemExtension#canPerformAction`。

要查询一个`ItemStack`是否能表现出一个特定的`ItemAbility`，调用`IItemStackExtension#canPerformAction`。注，这个对任意`Item`生效，不仅仅是对工具生效。

## 护甲
与工具相似，护甲使用一个等级系统（尽管是一个不同的系统）。工具系统中被称作`Tier`的东西，对于护甲而言称作`ArmorMaterial`。像上文一样，这个例子展示了如何添加铜护甲；可以按需适配。然而，不像`Tier`，`ArmorMaterial`需要被注册。对于原版的值，见`ArmorMaterial`类。

```Java
//ARMOR_MATERIALS是一个DeferredRegister<ArmorMaterial>

//我们将铜放在锁链和铁之间的某个位置。
public static final Holder<ArmorMaterial> COPPER_ARMOR_MATERIAL =
    ARMOR_MATERIALS.register("copper", () -> new ArmorMaterial(
        //决定这个护甲材料的防御值，依赖于它是护甲的哪一部分。
        Util.make(new EnumMap<>(ArmorItem.Type.class), map -> {
            map.put(ArmorItem.Type.BOOTS, 2);
            map.put(ArmorItem.Type.LEGGINGS, 4);
            map.put(ArmorItem.Type.CHESTPLATE, 6);
            map.put(ArmorItem.Type.HELMET, 2);
            map.put(ArmorItem.Type.BODY, 4);
        }),
        //决定等级的附魔能力。它代表了该护甲上的魔咒会有多好。
        //金使用25，我们将铜放在略低于它的位置。
        20,
        //决定穿上该护甲时的音效。
        //这个是被一个Holder包装。
        SoundEvents.ARMOR_EQUIP_GENERIC,
        //决定该护甲的修复物品。
        () -> Ingredient.of(Tags.Items.INGOTS_COPPER),
        //决定渲染时应用于护甲的纹理位置。
        //这也可以由覆写`IItemExtension#getArmorTexture`于你的物品来指定，如果护甲纹理需要更加动态。
        List.of(
            //创建一个新的护甲纹理，将会位于：
            //- 'assets/mod_id/textures/models/armor/copper_layer_1.png'给外层纹理
            //- 'assets/mod_id/textures/models/armor/copper_layer_2.png'给内层纹理（只有护腿）
            new ArmorMaterial.Layer(
                ResourceLocation.fromNamespaceAndPath(MOD_ID, "copper")
            ),
            //创建一个新的将被渲染在前面的纹理之上的护甲纹理，位于：
            //- 'assets/mod_id/textures/models/armor/copper_layer_1_overlay.png'给外层纹理
            //- 'assets/mod_id/textures/models/armor/copper_layer_2_overlay.png'给内层纹理（只有护腿）
            //`true`意味着该护甲材料是可染色的；然而，物品还必须被添加到`minecraft:dyeable`标签中。
            new ArmorMaterial.Layer(
                ResourceLocation.fromNamespaceAndPath(MOD_ID, "copper"), "_overlay", true
            )
        ),
        //返回护甲的韧性值，韧性值是一个额外的被包括在伤害计算中的值，更多信息见Minecraft Wiki的护甲运作的文章：
        //https://zh.minecraft.wiki/w/%E7%9B%94%E7%94%B2#%E6%8F%90%E4%BE%9B%E7%9B%94%E7%94%B2%E9%9F%A7%E6%80%A7
        //只有钻石和下界合金在这里有大于0的值，所以我们只是返回0。
        0,
        //返回护甲的击退抗性值。当穿着该护甲时，玩家在某种程度上免疫击退。如果玩家总共拥有1或更高的来自所有护甲部分组合起来的总击退抗性，他们完全不会受到任何击退。
        //只有下界合金在这里有大于0的值，所以我们只是返回0。
        0
    ));
```

接着，我们在物品注册中使用该护甲材料。

```Java
//ITEMS是一个DeferredRegister<Item>
public static final Supplier<ArmorItem> COPPER_HELMET = ITEMS.register("copper_helmet", () -> new ArmorItem(
        //所用的护甲材料。
        COPPER_ARMOR_MATERIAL,
        //所用的护甲类型。
        ArmorItem.Type.HELMET,
        //我们设置耐久度的物品属性。
        //ArmorItem.Type是一个有五个值的枚举：HELMET、CHESTPLATE、LEGGINGS、BOOTS和BODY。
        //BODY用于像狼或马的非玩家的实体。
        //原版的护甲材料通过使用一个基准值、再用一个特定于类型的常量相乘来决定它。
        //常量为BOOTS的13，LEGGINGS的15，CHESTPLATE的16，HELMET的11以及BODY的16。
        //如果我们不想使用这些比例，我们可以平常地设置耐久值。
        new Item.Properties().durability(ArmorItem.Type.HELMET.getDurability(15))
));
public static final Supplier<ArmorItem> COPPER_CHESTPLATE = ITEMS.register("copper_chestplate", () -> new ArmorItem(...));
public static final Supplier<ArmorItem> COPPER_LEGGINGS = ITEMS.register("copper_leggings", () -> new ArmorItem(...));
public static final Supplier<ArmorItem> COPPER_BOOTS = ITEMS.register("copper_boots", () -> new ArmorItem(...));
```

当创建你的护甲纹理时，在原版护甲纹理的基础上制作以看看哪个部分对应哪里是一个好主意。
