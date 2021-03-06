
> * 原文地址：[Exploring Kotlin’s hidden costs — Part 3](https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-3-3bf6e0dbf0a4)
> * 原文作者：[Christophe B.](https://medium.com/@BladeCoder)
> * 译文出自：[掘金翻译计划](https://github.com/xitu/gold-miner)
> * 本文永久链接：[https://github.com/xitu/gold-miner/blob/master/TODO/exploring-kotlins-hidden-costs-part-3.md](https://github.com/xitu/gold-miner/blob/master/TODO/exploring-kotlins-hidden-costs-part-3.md)
> * 译者：
> * 校对者：

# Exploring Kotlin’s hidden costs — Part 3

---

## Delegated properties and ranges

Following the overwhelming positive feedback I received after publishing the first two parts of this series about the Kotlin programming language, including a mention by Jake Wharton himself, I’m happy to continue the investigation. Don’t miss [part 1](https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-1-fbb9935d9b62) and [part 2](https://medium.com/@BladeCoder/exploring-kotlins-hidden-costs-part-2-324a4a50b70).

In this part 3, we’ll reveal more secrets of the Kotlin compiler and provide new tips to write more efficient code.

![](https://cdn-images-1.medium.com/max/800/1*-iKupZ7diZBEzTw87Bkaxg.jpeg)

### Delegated properties

A [delegated property](https://kotlinlang.org/docs/reference/delegated-properties.html) is a [property](https://kotlinlang.org/docs/reference/properties.html) whose getter and optional setter implementations are provided by an external object called the *delegate*. This allows reusable custom property implementations.

```
class Example {
    var p: String by Delegate()
}
```

The delegate object has to implement an `**operator**``getValue()` function, as well as a `setValue()` function for read/write properties. These functions will receive as extra arguments the **containing object instance** as well as the **property metadata** (like its name).

When a class declares a delegated property, code matching the following Java representation is generated by the compiler:

```
public final class Example {
   @NotNull
   private final Delegate p$delegate = new Delegate();
   // $FF: synthetic field
   static final KProperty[] $$delegatedProperties = new KProperty[]{(KProperty)Reflection.mutableProperty1(new MutablePropertyReference1Impl(Reflection.getOrCreateKotlinClass(Example.class), "p", "getP()Ljava/lang/String;"))};

   @NotNull
   public final String getP() {
      return this.p$delegate.getValue(this, $$delegatedProperties[0]);
   }

   public final void setP(@NotNull String var1) {
      Intrinsics.checkParameterIsNotNull(var1, "<set-?>");
      this.p$delegate.setValue(this, $$delegatedProperties[0], var1);
   }
}
```

Some static property metadata is added to the class. The delegate is initialized in the class constructor and is then invoked every time the property is read or written to.

#### Delegate instances

In the above example, **a new delegate instance is created** to implement the property. This is required when the delegate implementation is **stateful**, for example if it caches the computed value of the property locally:

```
class StringDelegate {
    private var cache: String? = null

    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        var result = cache
        if (result == null) {
            result = someOperation()
            cache = result
        }
        return result
    }
}
```

It’s also required to create a new delegate instance if it requires **extra parameters**, passed through its constructor:

```
class Example {
    private val nameView by BindViewDelegate<TextView>(R.id.name)
}
```

But there are some cases **where only a single delegate instance is required to implement any property**: when the delegate is stateless and the only variables it needs to do its job are the containing object instance and the property name, which are already provided. In that case, you can make the delegate a **singleton** by declaring it as `**object**` instead of `**class**`.

For example, the following singleton delegate retrieves the `Fragment` whose tag name matches the property name inside an Android `Activity`:

```
object FragmentDelegate {
    operator fun getValue(thisRef: Activity, property: KProperty<*>): Fragment? {
        return thisRef.fragmentManager.findFragmentByTag(property.name)
    }
}
```

Similarly, **any existing object can be extended to become a delegate**. `getValue()` and `setValue()` can also be declared as [**extension functions**](https://kotlinlang.org/docs/reference/extensions.html#extension-functions). Kotlin already provides built-in extension functions to allow using `Map` and `MutableMap` instances as delegates, using the property names as keys.

If you choose to reuse the same local delegate instance to implement multiple properties in the same class, you need to initialize this instance in the class constructor.

*Note*: since Kotlin 1.1, it’s also possible to declare [a local variable in a function as a delegated property](https://kotlinlang.org/docs/reference/delegated-properties.html#local-delegated-properties-since-11). In that case, the delegate can be initialized later, up to the point where the variable is declared in the function.

> Each delegated property declared in a class also involves **the overhead of its associated delegate object**, and adds some metadata to the class.
> Try to **reuse** delegates for different properties when it makes sense.
> Also consider whether delegated properties are your best option in cases where you would need to declare a large number of them.

#### Generic delegates

The delegate functions can be declared in a generic way, so the same delegate class can be used with various property types.

```
private var maxDelay: Long by SharedPreferencesDelegate<Long>()
```

However, if you use a generic delegate with a primitive type property like in the above example, **boxing and unboxing will occur** each time the property is read or written to, even if the declared primitive type is non-null.

> For delegated properties of non-null primitive types, prefer using **specialized delegate classes created for that specific value type** rather than a generic delegate, to avoid boxing overhead during each access to the property.

#### Standard delegates: lazy()

Kotlin provides a few standard delegates to cover common cases, like `[Delegates.notNull()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/not-null.html)`, `[Delegates.observable()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.properties/-delegates/observable.html)` and `[*lazy()*](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/lazy.html)`.

`**lazy()**` is a function returning a delegate for read-only properties that will evaluate the provided lambda to initialize the property value when first read.

```
private val dateFormat: DateFormat by lazy {
    SimpleDateFormat("dd-MM-yyyy", Locale.getDefault())
}
```

This is a neat way to **defer an expensive initialization** until it’s actually needed, improving performance while keeping the code readable.

It should be noted that `lazy()` is not an inline function and the lambda passed as argument will be compiled to a separate `Function` class and will not be inlined inside the returned delegate object.

What is often overlooked is that `**lazy()**`** has an optional **`**mode**`** argument** to determine which of 3 different types of delegates will be returned:

```
public fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)
public fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
        when (mode) {
            LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
            LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
            LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
        }
```

The default mode, `LazyThreadSafetyMode.**SYNCHRONIZED**`, will perform a relatively expensive **double-checked lock**, which is required to guarantee that the initialization block will run safely once when the property can be read from **multiple threads**.

If you know that a property is only going to be accessed from **a single thread** (like the main thread), then **locking can be avoided entirely** by explicitly using `LazyThreadSafetyMode.**NONE**` instead:

```
val dateFormat: DateFormat by lazy(LazyThreadSafetyMode.NONE) {
    SimpleDateFormat("dd-MM-yyyy", Locale.getDefault())
}
```

> Use the `lazy()` delegate to defer expensive initializations, and specify the thread safety mode to avoid locking when it’s not needed.

---

### Ranges

[Ranges](https://kotlinlang.org/docs/reference/ranges.html) are special expressions used to represent a finite set of values in Kotlin. The values can be of any `Comparable` type. These expressions are formed with functions which create objects implementing the `[ClosedRange](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/-closed-range/)` interface. The main function used to create a range is the `..` operator function.

#### Inclusion tests

The main purpose of a range expression is to write inclusion or exclusion tests using the `**in**` and `**!in**` operators.

```
if (i in 1..10) {
    println(i)
}
```

The implementation is optimized for **non-null primitive type ranges** (bounded by`Int`, `Long`, `Byte`, `Short`, `Float`, `Double` or `Char` values), so that the above example effectively gets compiled to this:

```
if(1 <= i && i <= 10) {
   System.out.println(i);
}
```

There is **zero overhead** and no extra object allocation. Range tests can also be included in a `**when**` expression.

```
val message = when (statusCode) {
    in 200..299 -> "OK"
    in 300..399 -> "Find it somewhere else"
    else -> "Oops"
}
```

This makes the code more readable than a series of `if{...} else if{...}` statements and is just as efficient.

However, **ranges have a small cost when there is at least one level of indirection** between their declaration and their usage in an inclusion test. Consider the following Kotlin code:

```
private val myRange get() = 1..10

fun rangeTest(i: Int) {
    if (i in myRange) {
        println(i)
    }
}
```

It involves the extra creation of an `IntRange` object after compilation:

```
private final IntRange getMyRange() {
   return new IntRange(1, 10);
}

public final void rangeTest(int i) {
   if(this.getMyRange().contains(i)) {
      System.out.println(i);
   }
}
```

Declaring the property getter as an `**inline**` function does not prevent the creation of this object either. *This is a case where the Kotlin 1.1 compiler could be improved. *At least, no boxing is involved to compare primitive types thanks to these specialized range classes.

> Try to declare non-null primitive type ranges **directly in the tests where they are used** with no indirection, to avoid the extra allocation of range objects.
> Alternatively, you can declare them as **constants** and reuse them.

Ranges can also be used with any other non-primitive `Comparable` type.

```
if (name in "Alfred".."Alicia") {
    println(name)
}
```

In that case, the implementation is not optimized and a `ClosedRange` object is always created as you can see after compilation:

```
if(RangesKt.rangeTo((Comparable)"Alfred", (Comparable)"Alicia")
   .contains((Comparable)name)) {
   System.out.println(name);
}
```

> If you need frequent inclusion tests against a range of non-primitive `Comparable` values, consider declaring this range as a constant to avoid repeated allocations of range objects.

#### Iterations: for loops

**Integral type ranges** (ranges of any primitive type but `Float` or `Double`) are also *progressions*: **they can be iterated over**. This allows to replace the classic Java `**for**` loop with a shorter syntax.

```
for (i in 1..10) {
    println(i)
}
```

This gets compiled to comparable optimized code with **zero overhead**:

```
int i = 1;
byte var3 = 10;
if(i <= var3) {
   while(true) {
      System.out.println(i);
      if(i == var3) {
         break;
      }
      ++i;
   }
}
```

To iterate *backwards*, you use the `[downTo()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/down-to.html)` infix function instead of `..`

```
for (i in 10 downTo 1) {
    println(i)
}
```

Again, there is zero overhead after compilation with this construct:

```
int i = 10;
byte var3 = 1;
if(i >= var3) {
   while(true) {
      System.out.println(i);
      if(i == var3) {
         break;
      }
      --i;
   }
}
```

However, **other iteration variants are not as well optimized**.

Here is another way to iterate backwards and produce the exact same result, using the `[reversed()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/reversed.html)` function combined with a range.

```
for (i in (1..10).reversed()) {
    println(i)
}
```

The resulting compiled code is unfortunately less pretty:

```
IntProgression var10000 = RangesKt.reversed((IntProgression)(new IntRange(1, 10)));
int i = var10000.getFirst();
int var3 = var10000.getLast();
int var4 = var10000.getStep();
if(var4 > 0) {
   if(i > var3) {
      return;
   }
} else if(i < var3) {
   return;
}

while(true) {
   System.out.println(i);
   if(i == var3) {
      return;
   }

   i += var4;
}
```

A temporary `IntRange` object is created to represent the range, then a second `IntProgression` object is created to reverse the values of the first one.

In fact, **any combination of more than one function to create a progression** generates similar code involving the small overhead of **creating at least two lightweight progression objects**.

This rule also applies to using the `[step()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/step.html)` infix function to modify the step of a progression, **even for a step of 1**:

```
for (i in 1..10 step 2) {
    println(i)
}
```

As a side note, when the generated code reads the `**last**` property of the `IntProgression`, this will perform a small computation to determine the exact last value of the range by taking the bounds and the step into consideration. In the above example, that last value would be 9.

Finally, there is also the useful `[until()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/until.html)` infix function to iterate up to, but excluding the upper value.

```
for (i in 0 until size) {
    println(i)
}
```

Sadly, **the compiler does not optimize this expression as well as a classic inclusive range**, and the iteration will also involve the creation of a range object:

```
IntRange var10000 = RangesKt.until(0, size);
int i = var10000.getFirst();
int var1 = var10000.getLast();
if(i <= var1) {
   while(true) {
      System.out.println(i);
      if(i == var1) {
         break;
      }
      ++i;
   }
}
```

*This is another area where the Kotlin 1.1 compiler could be improved.
*In the meantime, to get lighter compiled code, you can simply write:

```
for (i in 0..size - 1) {
    println(i)
}
```

> To iterate in a `**for**` loop, prefer using a range expression involving **a single function call to **`**..**`** or **`**downTo()**` to avoid the overhead of creating temporary progression objects.

#### Iterations: forEach()

Instead of using a `**for**` loop, it could be tempting to use the `forEach()` inline extension function on a range to obtain similar results.

```
(1..10).forEach {
    println(it)
}
```

But if you take a closer look at the signature of the `forEach()` function used here, you’ll notice that it’s not optimized for ranges but only for `Iterable`, so it requires the creation of an iterator. Here is the Java representation of the compiled code:

```
Iterable $receiver$iv = (Iterable)(new IntRange(1, 10));
Iterator var1 = $receiver$iv.iterator();

while(var1.hasNext()) {
   int element$iv = ((IntIterator)var1).nextInt();
   System.out.println(element$iv);
}
```

This code is **even less efficient** than the previous examples, because in addition to the creation of an `IntRange` object, you also have to pay the cost of an `IntIterator`. At least, this one generates primitive values.

> To iterate on a range, prefer using a simple `**for**` loop rather than calling the `forEach()` function on it, to avoid the overhead of an iterator object.

#### Iterations: collection indices

The Kotlin standard library provides a built-in `[*indices*](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/indices.html)` extension property to generate ranges for array indices and `Collection` indices.

```
val list = listOf("A", "B", "C")
for (i in list.indices) {
    println(list[i])
}
```

Surprisingly, iterating over this `*indices*`**gets compiled to optimized code as well**:

```
List list = CollectionsKt.listOf(new String[]{"A", "B", "C"});
int i = 0;
int var2 = ((Collection)list).size() - 1;
if(i <= var2) {
   while(true) {
      Object var3 = list.get(i);
      System.out.println(var3);
      if(i == var2) {
         break;
      }
      ++i;
   }
}
```

Here we can see no `IntRange` object being created at all and the list iteration is as efficient as it could be.

This works well for arrays and classes implementing `Collection`, so you could be tempted to roll your own `*indices*` extension property for custom classes while expecting the same iteration performance.

```
inline val SparseArray<*>.indices: IntRange
    get() = 0..size() - 1

fun printValues(map: SparseArray<String>) {
    for (i in map.indices) {
        println(map.valueAt(i))
    }
}
```

After compilation however, we can see that **this is not as efficient** because the compiler can not smartly avoid the creation of a range object:

```
public static final void printValues(@NotNull SparseArray map) {
   Intrinsics.checkParameterIsNotNull(map, "map");
   IntRange var10002 = new IntRange(0, map.size() - 1);
   int i = var10002.getFirst();
   int var2 = var10002.getLast();
   if(i <= var2) {
      while(true) {
         Object $receiver$iv = map.valueAt(i);
         System.out.println($receiver$iv);
         if(i == var2) {
            break;
         }
         ++i;
      }
   }
}
```

Instead, I would recommend that you limit yourself to implementing a custom `*lastIndex*` extension property:

```
inline val SparseArray<*>.lastIndex: Int
    get() = size() - 1

fun printValues(map: SparseArray<String>) {
    for (i in 0..map.lastIndex) {
        println(map.valueAt(i))
    }
}
```

> When iterating over **custom collections** not implementing the `Collection` interface, prefer **writing your own range of indices directly in the **`**for**`** loop** rather than relying on a function or property to generate the range, to avoid the allocation of a range object.

---

I hope this was as interesting for you to read as it was for me to write. You may expect more at a later date, but these 3 first parts cover everything I planned to write about initially. Please share if you like. Thank you!


---

> [掘金翻译计划](https://github.com/xitu/gold-miner) 是一个翻译优质互联网技术文章的社区，文章来源为 [掘金](https://juejin.im) 上的英文分享文章。内容覆盖 [Android](https://github.com/xitu/gold-miner#android)、[iOS](https://github.com/xitu/gold-miner#ios)、[React](https://github.com/xitu/gold-miner#react)、[前端](https://github.com/xitu/gold-miner#前端)、[后端](https://github.com/xitu/gold-miner#后端)、[产品](https://github.com/xitu/gold-miner#产品)、[设计](https://github.com/xitu/gold-miner#设计) 等领域，想要查看更多优质译文请持续关注 [掘金翻译计划](https://github.com/xitu/gold-miner)。
