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
## Misc
## hashcode
## toString
