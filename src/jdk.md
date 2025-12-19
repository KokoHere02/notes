# jdk24源码
## 阅读顺序（暂定）
- 集合顶层设计 Iterable -> Collection -> List / Set / Queue -> Map 等等
- ArrayList 构造 -> add / remove -> 扩容 -> get/set -> Iterator -> subList 等等
- Hashmap LinkedHashMap TreeMap / TreeSet
  - hash put resize getNode treeifyBin
- Collections / 工具类
- Thread / Runnable / Callable synchronized
  - Thread构造 start run join interrupt
- ConcurrentHashMap
- Virtual Thread
  
## interface Iterable<T> 
- 实现该接口允许对象成为增强for语句（for-each）
- Iterator<T> iterator() 
  - 返回一个迭代器
- `default void forEach(Consumer<? super T> action)`
- ```java 
  default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }```
## Collection
- 是所有集合的根类，通过List，Set，Queue 等接口的实现间接实现。
  - Collection是用来约束实现者的
  - 4个设计原则 1.有序/无序 2.是否重复 3.是否有encounter order（逻辑顺序） 4.实现全部交给子接口
  - 构造约定，允许有一个无参构造和一个有参构造参数（Collection<? extends E>）
  - 接口里有的方法不一定就是能用 例：`List.of(1,2,3).add(4); // UnsupportedOperationException`
  - 是否运行null是子实现的自由Collection没有规定
  - Collection 默认不保证线程安全
  - 判断相等可以自行优化 先比hash 后比equals
  - View Collections 只做代理 例：使用subList得到的List 如果修改它那就是修改原List等于直接修改底层的List
  - unmodifiable 禁止结构修改 immutable 状态永远不变
  - 为什么不extends Serializable 接口无法保证泛型元素可序列化
  - 默认方法不会加锁，如果是线程安全的必须重写该方法
- `int size()` 返回Collection长度 如果超过了Integer.MAX_VALUE 则返回Integer.MAX_VALUE
- `boolean isEmpty();` 如果不包含任何元素就返回true 反之false
- `boolean contains(Object o);` 如果集合中有一个元素能使 Object.equale(target, o) = true 就返回真
- `Iterator<E> iterator();` 返回迭代器对象 对顺序没有保证（除非该集合是某个类的实例）
- `Object[] toArray();` 返回包含集合中的所有元素，如果该集合的实现保证了迭代子返回的顺序，该方法必须以相同的顺序返回元素，返回的类型的Object。返回的数组是安全的。因为该集合不会维护对它的应用。
- `<T> T[] toArray(T[] a);` 返回该集合指定类型的数组，如果集合符合指定的数组，则返回到数组中，否则返回一个新数组，如果数组的大小大于集合元素的长度多余的位置会为空
- `boolean add(E e);` 确保该集合指定的类型（可选）。如果该集合因调用而发送变化则返回true反之false。有些集合会拒绝添加空元素，而另一些则会对可添加的元素类型进行限制，如果因为其他原因拒绝添加某个元素则必须抛出异常
- `boolean remove(Object o);` 如果集合中有一个或多个元素能够使用Obejcts.equals(o, e) 则将第一个相同的元素移除
- `boolean containsAll(Collection<?> c);` 如果该集合包含给定集合中的所有元素则返回true
- `boolean addAll(Collection<? extends E> c);` 将指定的集合添加到该集合中，如果修改这个指定的集合则该操作是为定义的
- `boolean removeAll(Collection<?> c);` 移除该集合中所有包含在指定集合中的元素
- `default boolean removeIf(Predicate<? super E> filter);` 删除满足条件的元素
- `boolean retainAll(Collection<?> c);` 移除所有不包含指定集合的元素
- `void clear();` 移除所有该集合的元素
- `default Spliterator<E> spliterator()` 是StreamAPI的底层入口，默认基于Iterator创建一个延迟绑定、fail-fast、可估算大小的Spliterator；性能一般

### abstract class AbstractCollection<E> implements Collection<E> Collection的实现
- 这是集合接口的骨架，以最小实现该接口所需的功能，要实现不可修改的集合，只需要扩展该类并提供迭代器和size方法的实现。迭代器返回的迭代器必须实现hasNext和next，要实现可修改集合，必须覆盖add方法
- `public boolean contains(Object o)` 提供了一个简易判断是否包含的方法，支持null
-  `public Object[] toArray()` 创建一个Object[size()]的数组，从迭代器里获取顺序数据依次放入数组中，迭代器可能和size()大小不同步，最后就需要检查迭代器里是否还要多余的元素finishToArray()处理多预期的元素
-  `public boolean add(E e)` 不支持add的方法需要具体的子类去实现

### interface SequencedCollection<E> extends Collection<E>
- 有确定的顺序，支持两端操作，支持反向视图 JAVA21新增
- `SequencedCollection<E> reversed();` 反转视图
- `default void addFirst(E e)` 新增元素到头部 
- `default void addLast(E e)` 新增元素到尾部

#### interface List<E> extends SequencedCollection<E>
- `boolean isEmpty();` 列表没有元素返回true
- `boolean addAll(int index, Collection<? extends E> c);` 在指定索引添加Collection
- `boolean retainAll(Collection<?> c);` 将指定列表不包含的全部移除
#####  class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable
- ArrayList 是一个基于可变长度数组的List实现，支持随机访问，运行null，非线程安全
- `private static final int DEFAULT_CAPACITY = 10;` 默认容量10
- `private static final Object[] EMPTY_ELEMENTDATA = {};`  是一个共享的 0 长度数组，用来避免为真正“容量为 0”的 ArrayList 分配多余对象
- ` private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};` 是ArrayList为了延迟分配默认容量（10）而设计的语义空数组，用来区分 new ArrayList<>() 和 new ArrayList<>(0) 的行为
- `transient Object[] elementData;` 是ArrayList底层存储数组所有元素都存在这里
  - transient: 不会被java默认的序列化机制序列化
- `public ArrayList` 构造方法，如果传了参那就会new Object[params] 如果传的为空那就直接等于 EMPTY_ELEMENTDATA。不传参直接等于 DEFAULTCAPACITY_EMPTY_ELEMENTDATA
- `public void trimToSize()` 是ArrayList提供的显示内存收缩手段，通过重建底层数组，将capacity压缩到size，并在size为0的时候复用EMPTY_ELEMENTDATA
- ` public void ensureCapacity(int minCapacity)` 是ArrayList的显示容量控制方法，通过判断当前数组是否足够，是否默认数组，实现了高性能的动态扩容
- `private Object[] grow(int minCapacity)` 扩容 会判断老的容量是否大于0或者不等于默认数组就会进行扩容否则就创建默认指定大小的数组 oldCapacity >> 1 扩容容量当前的一半 1.5倍
- `public boolean equals(Object o)` 判断2个List是否相等，支持各种List，并且检查并判断modCount有没有并发的被修改，如果被修改了就会抛出异常 是fail-fast设计的一部分
- `int hashCodeRange(int from, int to)` 计算List内from 到 to 范围内的hash 
- `public ListIterator<E> listIterator(int index)` 从指定的索引获取到迭代器对象
###### private class Itr implements Iterator<E>
- 是Iterator的内部优化实现，用于快速、fail-fast便利ArrayList
- `int cursor; ` 下一个next()返回的索引
- `int lastRet = -1; ` 上一个返回的索引
- `int expectedModCount;` 创建迭代器时的modCount，用于fail-fast
###### private class ListItr extends Itr implements ListIterator<E>
- 只能在ArrayList内部使用，支持双向遍历，索引访问元素插入，元素替换
- `public boolean hasPrevious()` 判断是否还有前一个元素返回
- `public int nextIndex()` 下一个索引
- `public int previousIndex()` 上一个索引
- `public E previous()` 获取下一个元素，更新索引下标
###### private static class SubList<E> extends AbstractList<E> implements RandomAccess
- ArrayList的内部类 用于实现subList的子列表，实现了大部分List的方法，减少内存开销，支持fail-fast
- `public Spliterator<E> spliterator()` 是为stream API 和并行遍历提供支持
###### final class ArrayListSpliterator implements Spliterator<E>
- 是ArrAyList内部高效Spliterator实现支持顺序遍历、并行拆分、fail-fast
- `private int index;` 当前索引
- `private int fence;` 结束索引
- `private int expectedModCount;` fail-fast版本检测
- ` private int getFence()` 如果fence < 0就初始化fence = size 并返回fence
-  `public ArrayListSpliterator trySplit()` 尝试分开 获取结束索引、开始索引、中间索引，如果开始索引大于中间索引直接返回空否则就分开
-  `public boolean tryAdvance(Consumer<? super E> action)` 尝试操作如果action=null 就直接结束
- `public void forEachRemaining(Consumer<? super E> action)` 对于每个剩余的元素进行操作
- `public long estimateSize()` 估计操作的元素 结束索引 - 开始索引