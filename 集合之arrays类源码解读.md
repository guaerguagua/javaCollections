jdk版本：1.8

arrays类提供了对集合类的一些常用操作，比如排序、转化为线程安全的集合类

## 排序
对于基本的数据类型都提供了sort函数，就以int为例子

```java
//对全量进行快排，快排的原理很简单，就是选用了一个基准量每次把比基准量大和小的分到两边，基于分治的思想来处理。java实现的快排做了很多优化，不是最朴素的快排，具体快排的实现以后有时间分析。
public static void sort(int[] a) {
        DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
    }
//对范围进行快排
public static void sort(int[] a, int fromIndex, int toIndex) {
        rangeCheck(a.length, fromIndex, toIndex);
        DualPivotQuicksort.sort(a, fromIndex, toIndex - 1, null, 0, 0);
    }
```
在java1.8以后对排序做了优化，对于长度超过1<<<13的数组切分成小段的进行快排，然后合并
```java
public static void parallelSort(byte[] a) {
        int n = a.length, p, g;
        if (n <= MIN_ARRAY_SORT_GRAN ||
            (p = ForkJoinPool.getCommonPoolParallelism()) == 1)
            DualPivotQuicksort.sort(a, 0, n - 1);
        else
            new ArraysParallelSortHelpers.FJByte.Sorter
                (null, a, new byte[n], 0, n, 0,
                 ((g = n / (p << 2)) <= MIN_ARRAY_SORT_GRAN) ?
                 MIN_ARRAY_SORT_GRAN : g).invoke();
    }
```
对于object类型的排序不是使用的快排，而是使用的merge+insert排序的方式，在1.8之后引入了timsort，默认使用timsort，可以通过参数配置使用mergesort的版本，注释里面说了之后的版本将只保留
timsort，Timsort是结合了合并排序（merge sort）和插入排序（insertion sort）而得出的排序算法，它在现实中有很好的效率。Tim Peters在2002年设计了该算法并在Python中使用（TimSort 是 Python 中 list.sort 的默认实现），以后再说具体原理。
```java
public static void sort(Object[] a) {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a);
        else
            ComparableTimSort.sort(a, 0, a.length, null, 0, 0);
}
/** To be removed in a future release. */
private static void legacyMergeSort(Object[] a) {
    Object[] aux = a.clone();
    mergeSort(aux, a, 0, a.length, 0);
}
private static void mergeSort(Object[] src,
                                  Object[] dest,
                                  int low,
                                  int high,
                                  int off) {
        int length = high - low;

        // Insertion sort on smallest arrays
        if (length < INSERTIONSORT_THRESHOLD) {
            for (int i=low; i<high; i++)
                for (int j=i; j>low &&
                         ((Comparable) dest[j-1]).compareTo(dest[j])>0; j--)
                    swap(dest, j, j-1);
            return;
        }

        // Recursively sort halves of dest into src
        int destLow  = low;
        int destHigh = high;
        low  += off;
        high += off;
        int mid = (low + high) >>> 1;
        mergeSort(dest, src, low, mid, -off);
        mergeSort(dest, src, mid, high, -off);

        // If list is already sorted, just copy from src to dest.  This is an
        // optimization that results in faster sorts for nearly ordered lists.
        if (((Comparable)src[mid-1]).compareTo(src[mid]) <= 0) {
            System.arraycopy(src, low, dest, destLow, length);
            return;
        }

        // Merge sorted halves (now in src) into dest
        for(int i = destLow, p = low, q = mid; i < destHigh; i++) {
            if (q >= high || p < mid && ((Comparable)src[p]).compareTo(src[q])<=0)
                dest[i] = src[p++];
            else
                dest[i] = src[q++];
        }
    }

    /**
     * Swaps x[a] with x[b].
     */
    private static void swap(Object[] x, int a, int b) {
        Object t = x[a];
        x[a] = x[b];
        x[b] = t;
    }
```
## 并行前缀操作
对集合中的元素做一些操作，比如加法什么的，针对大集合通过并行的方式会比循环更高效
```java 
public static <T> void parallelPrefix(T[] array, BinaryOperator<T> op) {
        Objects.requireNonNull(op);
        if (array.length > 0)
            new ArrayPrefixHelpers.CumulateTask<>
                    (null, op, array, 0, array.length).invoke();
    }
 ```
## 二分查找
对于基本数据类型，以int为例子，可以指定查找的范围。
```java
public static int binarySearch(int[] a, int fromIndex, int toIndex,
                                   int key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}
private static int binarySearch0(int[] a, int fromIndex, int toIndex,
                                     int key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        int midVal = a[mid];

        if (midVal < key)
            low = mid + 1;
        else if (midVal > key)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}
```
对于object类型也是同理，不过比较的时候通过comparable接口
```java
public static int binarySearch(Object[] a, int fromIndex, int toIndex,
                                   Object key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(Object[] a, int fromIndex, int toIndex,
                                 Object key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        @SuppressWarnings("rawtypes")
        Comparable midVal = (Comparable)a[mid];
        @SuppressWarnings("unchecked")
        int cmp = midVal.compareTo(key);

        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}
```
## 判断两个数组是否相等
基础数据类型==比较，float和double用int，long表示以后==比较，object调用equals方法比较。
```java
public static boolean equals(int[] a, int[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (a[i] != a2[i])
            return false;

    return true;
}
public static boolean equals(double[] a, double[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (Double.doubleToLongBits(a[i])!=Double.doubleToLongBits(a2[i]))
            return false;

    return true;
}
public static boolean equals(Object[] a, Object[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++) {
        Object o1 = a[i];
        Object o2 = a2[i];
        if (!(o1==null ? o2==null : o1.equals(o2)))
            return false;
    }

    return true;
}
```
## 填充
```java
public static void fill(Object[] a, int fromIndex, int toIndex, Object val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}
public static void fill(int[] a, int fromIndex, int toIndex, int val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}
```
## 复制克隆
基本类型
```java
public static int[] copyOf(int[] original, int newLength) {
    int[] copy = new int[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
```
object
```java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
//Array中的newInstance方法
public static Object newInstance(Class<?> componentType, int length)
    throws NegativeArraySizeException {
    return newArray(componentType, length);
}
//调用本地方法
private static native Object newArray(Class<?> componentType, int length)
    throws NegativeArraySizeException;
```
## asList
这个方法把一个可变参数或者object数组转化为list集合的对象，可以使用list的api来操作数组，但是有一点需要特别注意，这个list是不可变长度的，使用add和remove方法都会抛出
UnsupportOperationException，这个是因为这个list不是调用的java默认的arrayList来实现的，而是使用Arrays内部的static内部类来实现的。直接看代码
```java
public static <T> List<T> asList(T... a) {
        return new ArrayList<>(a);
    }

    /**
     * @serial include
     */
    private static class ArrayList<E> extends AbstractList<E>
        implements RandomAccess, java.io.Serializable
    {
        private static final long serialVersionUID = -2764017481108945198L;
        private final E[] a;

        ArrayList(E[] array) {
            a = Objects.requireNonNull(array);
        }

        @Override
        public int size() {
            return a.length;
        }

        @Override
        public Object[] toArray() {
            return a.clone();
        }

        @Override
        @SuppressWarnings("unchecked")
        public <T> T[] toArray(T[] a) {
            int size = size();
            if (a.length < size)
                return Arrays.copyOf(this.a, size,
                                     (Class<? extends T[]>) a.getClass());
            System.arraycopy(this.a, 0, a, 0, size);
            if (a.length > size)
                a[size] = null;
            return a;
        }

        @Override
        public E get(int index) {
            return a[index];
        }

        @Override
        public E set(int index, E element) {
            E oldValue = a[index];
            a[index] = element;
            return oldValue;
        }

        @Override
        public int indexOf(Object o) {
            E[] a = this.a;
            if (o == null) {
                for (int i = 0; i < a.length; i++)
                    if (a[i] == null)
                        return i;
            } else {
                for (int i = 0; i < a.length; i++)
                    if (o.equals(a[i]))
                        return i;
            }
            return -1;
        }

        @Override
        public boolean contains(Object o) {
            return indexOf(o) != -1;
        }

        @Override
        public Spliterator<E> spliterator() {
            return Spliterators.spliterator(a, Spliterator.ORDERED);
        }

        @Override
        public void forEach(Consumer<? super E> action) {
            Objects.requireNonNull(action);
            for (E e : a) {
                action.accept(e);
            }
        }

        @Override
        public void replaceAll(UnaryOperator<E> operator) {
            Objects.requireNonNull(operator);
            E[] a = this.a;
            for (int i = 0; i < a.length; i++) {
                a[i] = operator.apply(a[i]);
            }
        }

        @Override
        public void sort(Comparator<? super E> c) {
            Arrays.sort(a, c);
        }
    }
```
这里面并没有重写add和remove方法，所以调用的时候会调用AbstractList的add和remove方法，直接抛出异常。猜猜看下面的输出是什么？
```java
public static void main(String[] args) {
    Integer[] integers = new Integer[]{1,2};
    List<Integer> list =Arrays.asList(integers);
    for(Integer i:list){
        System.out.println(i);
    }
    int[] int_arr = new int[]{1,2,3};
    for(Object i :Arrays.asList(int_arr)){
        System.out.println(i);
    }

    List<Object> list1 = Arrays.asList(1,2,3);
    for(Object i:list1){
        System.out.println(i);
    }
    Scanner sc = new Scanner(System.in);
    while (sc.hasNext()) {
    }
}
输出：
1
2
[I@4554617c
1
2
3
```
## hashcode
对于object数组
```java
public static int hashCode(Object a[]) {
        if (a == null)
            return 0;

        int result = 1;

        for (Object element : a)
            result = 31 * result + (element == null ? 0 : element.hashCode());

        return result;
    }
```
基本类型，比如
```java
public static int hashCode(double a[]) {
        if (a == null)
            return 0;

        int result = 1;
        for (double element : a) {
            long bits = Double.doubleToLongBits(element);
            result = 31 * result + (int)(bits ^ (bits >>> 32));
        }
        return result;
    }
```
还有deepHashCode，就是如果object数组的元素也是数组类型，再递归调用方法计算下一层的hash值。
```java
public static int deepHashCode(Object a[]) {
    if (a == null)
        return 0;

    int result = 1;

    for (Object element : a) {
        int elementHash = 0;
        if (element instanceof Object[])
            elementHash = deepHashCode((Object[]) element);
        else if (element instanceof byte[])
            elementHash = hashCode((byte[]) element);
        else if (element instanceof short[])
            elementHash = hashCode((short[]) element);
        else if (element instanceof int[])
            elementHash = hashCode((int[]) element);
        else if (element instanceof long[])
            elementHash = hashCode((long[]) element);
        else if (element instanceof char[])
            elementHash = hashCode((char[]) element);
        else if (element instanceof float[])
            elementHash = hashCode((float[]) element);
        else if (element instanceof double[])
            elementHash = hashCode((double[]) element);
        else if (element instanceof boolean[])
            elementHash = hashCode((boolean[]) element);
        else if (element != null)
            elementHash = element.hashCode();

        result = 31 * result + elementHash;
    }

    return result;
}
```
## toString
```java
public static String toString(int[] a) {
    if (a == null)
        return "null";
    int iMax = a.length - 1;
    if (iMax == -1)
        return "[]";

    StringBuilder b = new StringBuilder();
    b.append('[');
    for (int i = 0; ; i++) {
        b.append(a[i]);
        if (i == iMax)
            return b.append(']').toString();
        b.append(", ");
    }
}
public static String toString(Object[] a) {
    if (a == null)
        return "null";

    int iMax = a.length - 1;
    if (iMax == -1)
        return "[]";

    StringBuilder b = new StringBuilder();
    b.append('[');
    for (int i = 0; ; i++) {
        b.append(String.valueOf(a[i]));
        if (i == iMax)
            return b.append(']').toString();
        b.append(", ");
    }
}
```
## deepToString
深度toString，如果object数组是多维数组，可以递归打出多层的信息，遇到循环引用会判断。
```java
public static String deepToString(Object[] a) {
    if (a == null)
        return "null";

    int bufLen = 20 * a.length;
    if (a.length != 0 && bufLen <= 0)
        bufLen = Integer.MAX_VALUE;
    StringBuilder buf = new StringBuilder(bufLen);
    deepToString(a, buf, new HashSet<Object[]>());
    return buf.toString();
}

private static void deepToString(Object[] a, StringBuilder buf,
                                 Set<Object[]> dejaVu) {
    if (a == null) {
        buf.append("null");
        return;
    }
    int iMax = a.length - 1;
    if (iMax == -1) {
        buf.append("[]");
        return;
    }

    dejaVu.add(a);
    buf.append('[');
    for (int i = 0; ; i++) {

        Object element = a[i];
        if (element == null) {
            buf.append("null");
        } else {
            Class<?> eClass = element.getClass();

            if (eClass.isArray()) {
                if (eClass == byte[].class)
                    buf.append(toString((byte[]) element));
                else if (eClass == short[].class)
                    buf.append(toString((short[]) element));
                else if (eClass == int[].class)
                    buf.append(toString((int[]) element));
                else if (eClass == long[].class)
                    buf.append(toString((long[]) element));
                else if (eClass == char[].class)
                    buf.append(toString((char[]) element));
                else if (eClass == float[].class)
                    buf.append(toString((float[]) element));
                else if (eClass == double[].class)
                    buf.append(toString((double[]) element));
                else if (eClass == boolean[].class)
                    buf.append(toString((boolean[]) element));
                else { // element is an array of object references
                    if (dejaVu.contains(element))
                        buf.append("[...]");
                    else
                        deepToString((Object[])element, buf, dejaVu);
                }
            } else {  // element is non-null and not an array
                buf.append(element.toString());
            }
        }
        if (i == iMax)
            break;
        buf.append(", ");
    }
    buf.append(']');
    dejaVu.remove(a);
}
```
## deepEquals
deepEquals深度比较两个数组是否相等,这里和equals方法的区别目前看来在于实现方式，equals方法最后是通过object.equals(other)来比较两个object数组的element，这里是如果发现element
是object数组就递归调用比较，如果object[]的.equals方法实现了深度比较就没有这个递归调用比较的意思了，所以我猜测object的.equals方法实现只是比较了引用地址，所以才需要这个deepEquals。
那么简单来说equals适用于两个数组对象的元素都不是object数组的类型，deepEquals适用于多层数组深度比较的情况。
```java
public static boolean deepEquals(Object[] a1, Object[] a2) {
        if (a1 == a2)
            return true;
        if (a1 == null || a2==null)
            return false;
        int length = a1.length;
        if (a2.length != length)
            return false;

        for (int i = 0; i < length; i++) {
            Object e1 = a1[i];
            Object e2 = a2[i];

            if (e1 == e2)
                continue;
            if (e1 == null)
                return false;

            // Figure out whether the two elements are equal
            boolean eq = deepEquals0(e1, e2);

            if (!eq)
                return false;
        }
        return true;
    }

    static boolean deepEquals0(Object e1, Object e2) {
        assert e1 != null;
        boolean eq;
        if (e1 instanceof Object[] && e2 instanceof Object[])
            eq = deepEquals ((Object[]) e1, (Object[]) e2);
        else if (e1 instanceof byte[] && e2 instanceof byte[])
            eq = equals((byte[]) e1, (byte[]) e2);
        else if (e1 instanceof short[] && e2 instanceof short[])
            eq = equals((short[]) e1, (short[]) e2);
        else if (e1 instanceof int[] && e2 instanceof int[])
            eq = equals((int[]) e1, (int[]) e2);
        else if (e1 instanceof long[] && e2 instanceof long[])
            eq = equals((long[]) e1, (long[]) e2);
        else if (e1 instanceof char[] && e2 instanceof char[])
            eq = equals((char[]) e1, (char[]) e2);
        else if (e1 instanceof float[] && e2 instanceof float[])
            eq = equals((float[]) e1, (float[]) e2);
        else if (e1 instanceof double[] && e2 instanceof double[])
            eq = equals((double[]) e1, (double[]) e2);
        else if (e1 instanceof boolean[] && e2 instanceof boolean[])
            eq = equals((boolean[]) e1, (boolean[]) e2);
        else
            eq = e1.equals(e2);
        return eq;
    }
```
## setAll
1.8以后的方法
```java
    public static <T> void setAll(T[] array, IntFunction<? extends T> generator) {
        Objects.requireNonNull(generator);
        for (int i = 0; i < array.length; i++)
            array[i] = generator.apply(i);
    }
    //并行的，函数式编程表达
        public static <T> void parallelSetAll(T[] array, IntFunction<? extends T> generator) {
        Objects.requireNonNull(generator);
        IntStream.range(0, array.length).parallel().forEach(i -> { array[i] = generator.apply(i); });
    }
```
## stream
使用1.8以后的stream特性，这块没有研究，先放着，以后说。
```java
public static <T> Stream<T> stream(T[] array) {
        return stream(array, 0, array.length);
    }

    /**
     * Returns a sequential {@link Stream} with the specified range of the
     * specified array as its source.
     *
     * @param <T> the type of the array elements
     * @param array the array, assumed to be unmodified during use
     * @param startInclusive the first index to cover, inclusive
     * @param endExclusive index immediately past the last index to cover
     * @return a {@code Stream} for the array range
     * @throws ArrayIndexOutOfBoundsException if {@code startInclusive} is
     *         negative, {@code endExclusive} is less than
     *         {@code startInclusive}, or {@code endExclusive} is greater than
     *         the array size
     * @since 1.8
     */
    public static <T> Stream<T> stream(T[] array, int startInclusive, int endExclusive) {
        return StreamSupport.stream(spliterator(array, startInclusive, endExclusive), false);
    }
```
## spliterator
Spliterator（splitable iterator可分割迭代器）接口是Java为了并行遍历数据源中的元素而设计的迭代器，这个可以类比最早Java提供的顺序遍历迭代器Iterator，但一个是顺序遍历，一个是并行遍历

从最早Java提供顺序遍历迭代器Iterator时，那个时候还是单核时代，但现在多核时代下，顺序遍历已经不能满足需求了...如何把多个任务分配到不同核上并行执行，才是能最大发挥多核的能力，所以Spliterator应运而生啦

因为对于数据源而言...集合是描述它最多的情况，所以Java已经默认在集合框架中为所有的数据结构提供了一个默认的Spliterator实现，
相应的这个实现其实就是底层Stream如何并行遍历（Stream.isParallel()）的实现啦，因此平常用到Spliterator的情况是不多的...因为Java8这次正是一次引用函数式编程的思想，
你只需要告诉JDK你要做什么并行任务，关注业务本身，至于如何并行，怎么并行效率最高，就交给JDK自己去思考和优化速度了（想想以前写如何并发的代码被支配的恐惧吧）作为调用者我们
只需要去关心一些filter，map，collect等业务操作即可
参考[Java8里面的java.util.Spliterator接口有什么用？](https://segmentfault.com/q/1010000007087438)
```java
public static <T> Spliterator<T> spliterator(T[] array) {
        return Spliterators.spliterator(array,
                                        Spliterator.ORDERED | Spliterator.IMMUTABLE);
    }
```