数据组件是一个映射（map）中的键值对，用于存储`ItemStack`中的数据。每一块数据，如烟花爆炸或工具属性，被以真实的对象的形式存储在物品堆叠中，让这些值变得可见、可操作，而不需要去动态地转换一个总的被编码的实例（例如`CompoundTag`、`JsonElement`）。

## DataComponentType
每个数据组件有一个相关联的`DataComponentType<T>`，其中`T`是组件值的类型。`DataComponentType`代表了一个键，来引用存储着的组件值，协同一些Codec来处理向硬盘和网络读写数据，如果需要的话。

一个已经存在的数据组件（译注：类型）列表可以在`DataComponents`中找到。

### 创建自定义数据组件
和`DataComponentType`相关的组件值必须实现`hashCode`和`equals`，且存储时应当被视为不可变的。

> 注：组件值可以很容易地用一个record实现。record的字段是不可变的，且它实现了`hashCode`和`equals`。

```Java
//一个record例子
public record ExampleRecord(int value1, boolean value2) {}

//一个class例子
public class ExampleClass {

    private final int value1;
    //可以是可变的，但使用时要小心
    private boolean value2;

    public ExampleClass(int value1, boolean value2) {
        this.value1 = value1;
        this.value2 = value2;
    }

    @Override
    public int hashCode() {
        return Objects.hash(this.value1, this.value2);
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == this) {
            return true;
        } else {
            return obj instanceof ExampleClass ex
                && this.value1 == ex.value1
                && this.value2 == ex.value2;
        }
    }
}
```

一个标准的`DataComponentType`可以通过`DataComponentType#builder`创建，并通过`DataComponentType.Builder#build`构建。builder包含三项设置：`persistent`、`networkSynchronized`和`cacheEncoding`。

`persistent`指出用于将组件值读取和写入到磁盘中的`Codec`。`networkSynchronized`指出用于将组件值跨网络读写的`StreamCodec`。如果`networkSynchroized`未被指明，则`persistent`中提供的`Codec`会被包装，并被用作`StreamCodec`。

> 警告：在builder中，`persistent`和`networkSynchronized`必须至少有其一被提供；否则，一个`NullPointerException`会被抛出。如果没有需要跨网络传输的数据，那么将`networkSynchronized`设为`StreamCodec#unit`，提供默认的组件值。

`cacheEncoding`会缓存`Codec`的编码结果，若组件值未改变，则后续的所有编码操作都会使用该缓存值。它应当只被用在组件值被预期很少或从不改变的情况。

`DataComponentType`是注册对象，且必须被注册。

```Java
//使用ExampleRecord(int, boolean)
//只有一个Codec和/或StreamCodec应该被使用在下面
//出于示例，提供了多个

//基本的Codec
public static final Codec<ExampleRecord> BASIC_CODEC = RecordCodecBuilder.create(instance ->
    instance.group(
        Codec.INT.fieldOf("value1").forGetter(ExampleRecord::value1),
        Codec.BOOL.fieldOf("value2").forGetter(ExampleRecord::value2)
    ).apply(instance, ExampleRecord::new)
);
public static final StreamCodec<ByteBuf, ExampleRecord> BASIC_STREAM_CODEC = StreamCodec.composite(
    ByteBufCodecs.INT, ExampleRecord::value1,
    ByteBufCodecs.BOOL, ExampleRecord::value2,
    ExampleRecord::new
);

//Unit Stream Codec，如果没有数据需要跨网络传输
public static final StreamCodec<ByteBuf, ExampleRecord> UNIT_STREAM_CODEC = StreamCodec.unit(new ExampleRecord(0, false));


//在零一个类中
//特化的DeferredRegister.DataComponents简化了数据组件的注册，以及避免了一些`Supplier`中的`DataComponentType.Builder`的泛型推断问题
public static final DeferredRegister.DataComponents REGISTRAR = DeferredRegister.createDataComponents(Registries.DATA_COMPONENT_TYPE, "examplemod");

public static final DeferredHolder<DataComponentType<?>, DataComponentType<ExampleRecord>> BASIC_EXAMPLE = REGISTRAR.registerComponentType(
    "basic",
    builder -> builder
        //用于读取/写入数据到磁盘的Codec
        .persistent(BASIC_CODEC)
        //用于跨网络读写数据的Codec
        .networkSynchronized(BASIC_STREAM_CODEC)
);

//组件不会被保存到磁盘
public static final DeferredHolder<DataComponentType<?>, DataComponentType<ExampleRecord>> TRANSIENT_EXAMPLE = REGISTRAR.registerComponentType(
    "transient",
    builder -> builder.networkSynchronized(BASIC_STREAM_CODEC)
);

//没有数据会被跨网络同步
public static final DeferredHolder<DataComponentType<?>, DataComponentType<ExampleRecord>> NO_NETWORK_EXAMPLE = REGISTRAR.registerComponentType(
   "no_network",
   builder -> builder
        .persistent(BASIC_CODEC)
        //注，这里我们用一个单位StreamCodec
        .networkSynchronized(UNIT_STREAM_CODEC)
);
```

## 组件映射
所有数据组件都存储在一个`DataComponentMap`中，使用`DataComponentType`作为键，对象作为值。`DataComponentMap`在功能上类似一个只读的`Map`。因此，存在方法以获取（`#get`）根据`DataComponentType`获取一个条目，或在条目不存在时提供一个默认值（通过`getOrDefault`）。

```Java
//对于某个DataComponentMap map

//会获取到染料颜色，如果组件存在的话
//否则为null
@Nullable
DyeColor color = map.get(DataComponents.BASE_COLOR);
```

### PatchedDataComponentMap
由于默认的`DataComponentMap`只提供了用于读取操作的方法，写入数据的操作通过使用子类`PatchedDataComponentMap`来支持。包括设置（`#set`）一个组件的值或将其完全地移除（`#remove`）。

`PatchedDataComponentMap`使用一个原型和修订映射来存储变动。原型是一个包含了默认的组件及该map应该有的它们的值的`DataComponentMap`。修订映射是一个由`DataComponentType`到包含了对默认组件的变动的`Optional`值的映射。

```Java
//对于某个PatchedDataComponentMap map

//将基本颜色（Base Color）设置为白色
map.set(DataComponents.BASE_COLOR, DyeColor.WHITE);

//通过以下途径移除基本颜色（Base Color）：
// - 如果没有默认值被提供，移除相关修订
// - 如果存在默认值，设定一个空的Optional值
map.remove(DataComponents.BASE_COLOR);
```

> 危险：原型和修订映射都是`PatchedDataComponentMap`的哈希码的一部分。因此，映射中的任何组件值都应该被视为**不可变的**。在修改一个数据组件的值后，总应该调用`#set`或下文讨论的和其相关的方法之一。

## 组件持有者
所有能持有数据组件的实例实现了`DataComponentHolder`。`DataComponentHolder`事实上是一个`DataComponentMap`内部的只读方法的代理。

```Java
//对于某个ItemStack stack

//代理给'DataComponentMap#get'
@Nullable
DyeColor color = stack.get(DataComponents.BASE_COLOR);
```

### MutableDataComponentHolder
`MutableDataComponentHolder`是一个NeoForge提供的接口，用于支持对组件映射的写入方法。原版和NeoForge内部的所有实现使用一个`PatchedDataComponentMap`来存储数据组件，所以`#set`方法和`#remove`方法也有相同名称的代理方法。

此外，`MutableDataComponentHolder`还提供了一个`#update`方法，它处理：获取组件值或提供的默认值，如果没有值被设置了的话；对值进行操作；接着将其设置回映射当中。操作符要么是一个`UnaryOperator`，它接受组件值的输入并返回组件值；要么是一个`BiFunction`，它接受组件值和另外一个对象的输入并返回组件值。

```Java
//对于某个ItemStack stack

FireworkExplosion explosion = stack.get(DataComponents.FIREWORK_EXPLOSION);

//修改组件的值
explosion = explosion.withFadeColors(new IntArrayList(new int[] {1, 2, 3}));

//既然我们修改了组件的值，`set`应该在之后被调用
stack.set(DataComponents.FIREWORK_EXPLOSION, explosion);

//更新组件的值(会在内部调用`set`)
stack.update(
    DataComponents.FIREWORK_EXPLOSION,
    //默认值，如果组件的值不存在
    FireworkExplosion.DEFAULT,
    //返回一个新的FireworkExplosion对象来`set`
    explosion -> explosion.withFadeColors(new IntArrayList(new int[] {4, 5, 6}))
);

stack.update(
    DataComponents.FIREWORK_EXPLOSION,
    //默认值，如果组件的值不存在
    FireworkExplosion.DEFAULT,
    //一个提供给函数对象（Function）的对象
    new IntArrayList(new int[] {7, 8, 9}),
    //返回一个新的FireworkExplosion对象来`set`
    FireworkExplosion::withFadeColors
);
```

## 向物品添加默认的数据组件
尽管数据组件存储在`ItemStack`中，可以设置一个默认组件的映射到`Item`，它会将其传递给`ItemStack`于构造时，作为一个原型。可以通过`Item.Properties#component`来添加一个组件到`Item`。

```Java
//对于某个DeferredRegister.Items REGISTRAR
public static final Item COMPONENT_EXAMPLE = REGISTRAR.register("component",
    //register方法优先于其他重载被使用，因为DataComponentType尚未被注册
    () -> new Item(
        new Item.Properties()
        .component(BASIC_EXAMPLE.value(), new ExampleRecord(24, true))
    )
);
```

如果数据组件应当被添加到一个原版的或其他Mod的已有的物品上，那么应当在Mod事件总线上监听`ModifyDefaultComponentEvent`。该事件提供了`modify`和`modifyMatching`方法，允许相关联的物品的`DataComponentPatch.Builder`被修改。builder既可以设置（`#set`）组件，也可以移除（`#remove`）已经存在的组件。

```Java
@SubscribeEvent //在Mod事件总线上
public static void modifyComponents(ModifyDefaultComponentsEvent event) {
    //设置西瓜籽的组件
    event.modify(Items.MELON_SEEDS, builder ->
        builder.set(BASIC_EXAMPLE.value(), new ExampleRecord(10, false))
    );

    //移除任何有合成剩余物的物品的组件
    event.modifyMatching(
        item -> item.hasCraftingRemainingItem(),
        builder -> builder.remove(DataComponents.BUCKET_ENTITY_DATA)
    );
}
```

## 使用自定义组件持有者
要创建一个自定义的数据组件持有者，持有者对象仅需要实现`MutableDataComponentHolder`，以及实现缺失的方法。持有者对象必须包含一个代表`PatchedDataComponentMap`的字段来实现相关的方法。

```Java
public class ExampleHolder implements MutableDataComponentHolder {

    private int data;
    private final PatchedDataComponentMap components;

    //可以提供重载以提供映射本身
    public ExampleHolder() {
        this.data = 0;
        this.components = new PatchedDataComponentMap(DataComponentMap.EMPTY);
    }

    @Override
    public DataComponentMap getComponents() {
        return this.components;
    }

    @Nullable
    @Override
    public <T> T set(DataComponentType<? super T> componentType, @Nullable T value) {
        return this.components.set(componentType, value);
    }

    @Nullable
    @Override
    public <T> T remove(DataComponentType<? extends T> componentType) {
        return this.components.remove(componentType);
    }

    @Override
    public void applyComponents(DataComponentPatch patch) {
        this.components.applyPatch(patch);
    }

    @Override
    public void applyComponents(DataComponentMap components) {
        this.components.setAll(p_330402_);
    }

    //其他方法
}
```

### DataComponentPatch和Codec
要持久化组件或跨网络传输信息，（组件）持有者可以传输整个`DataComponentMap`。然而，这基本上是在浪费信息，因为任何默认值都将已经存在，无论数据被传输到哪里。所以，取而代之的是，我们用一个`DataComponentPatch`来传输相关的数据。`DataComponentPatch`只包含组件映射的修订信息，没有任何默认值。接着修订会在接收者的位置应用于原型。

一个`DataComponentPatch`可以通过`#patch`从一个`PatchedDataComponentMap`创建。类似地，`PatchedDataComponentMap#fromPatch`可以用给定的`DataComponentMap`原型和一个`DataComponentPatch`来构造一个`PatchedDataComponentMap`。

```Java
public class ExampleHolder implements MutableDataComponentHolder {

    public static final Codec<ExampleHolder> CODEC = RecordCodecBuilder.create(instance ->
        instance.group(
            Codec.INT.fieldOf("data").forGetter(ExampleHolder::getData),
            DataCopmonentPatch.CODEC.optionalFieldOf("components", DataComponentPatch.EMPTY).forGetter(holder -> holder.components.asPatch())
        ).apply(instance, ExampleHolder::new)
    );

    public static final StreamCodec<RegistryFriendlyByteBuf, ExampleHolder> STREAM_CODEC = StreamCodec.composite(
        ByteBufCodecs.INT, ExampleHolder::getData,
        DataComponentPatch.STREAM_CODEC, holder -> holder.components.asPatch(),
        ExampleHolder::new
    );

    // ...

    public ExampleHolder(int data, DataComponentPatch patch) {
        this.data = data;
        this.components = PatchedDataComponentMap.fromPatch(
            // The prototype map to apply to
            DataComponentMap.EMPTY,
            // The associated patches
            patch
        );
    }

    // ...
}
```

跨网络同步（组件）持有者的数据和向磁盘读写数据必须被手动完成。
