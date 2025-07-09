# 桶排序 (Bucket Sort)

桶排序是一种分布式排序算法，它将元素分布到有限数量的桶中，每个桶再单独排序（可以使用别的排序算法或递归地继续使用桶排序）。

## 基本思想

1. 设置一个定量的数组当作空桶
2. 遍历原始数组，将元素按照一定规则映射到对应的桶中
3. 对每个非空的桶中的元素进行排序（可以使用任何排序算法）
4. 按顺序访问桶，将桶中元素合并得到排序结果

## 图解过程

![桶排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/bucket-sort.png)

## 代码实现

### 浮点数桶排序

下面是一个用于排序0.0-1.0范围内浮点数的桶排序实现：

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class BucketSort {

    public static void bucketSort(float[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        // 创建桶并初始化
        int n = arr.length;
        List<Float>[] buckets = new ArrayList[n];
        for (int i = 0; i < n; i++) {
            buckets[i] = new ArrayList<>();
        }
        
        // 将元素放入桶中
        for (float item : arr) {
            int bucketIndex = (int) (item * n); // 计算元素应该放入哪个桶
            buckets[bucketIndex].add(item);
        }
        
        // 对每个桶内的元素排序
        for (List<Float> bucket : buckets) {
            Collections.sort(bucket);
        }
        
        // 合并桶内元素
        int index = 0;
        for (List<Float> bucket : buckets) {
            for (float item : bucket) {
                arr[index++] = item;
            }
        }
    }
    
    // 测试
    public static void main(String[] args) {
        float[] arr = {0.42f, 0.32f, 0.33f, 0.52f, 0.37f, 0.47f, 0.51f, 0.78f, 0.21f};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        bucketSort(arr);
        
        System.out.println("排序后数组：");
        printArray(arr);
    }
    
    // 打印数组
    public static void printArray(float[] arr) {
        for (float i : arr) {
            System.out.print(i + " ");
        }
        System.out.println();
    }
}
```

### 整数桶排序

对于整数序列，我们可以根据其范围确定桶的数量和映射规则：

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class IntegerBucketSort {

    public static void bucketSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        // 找出最大值和最小值
        int max = arr[0];
        int min = arr[0];
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] > max) {
                max = arr[i];
            }
            if (arr[i] < min) {
                min = arr[i];
            }
        }
        
        // 计算桶的数量
        int bucketCount = (max - min) / arr.length + 1;
        List<Integer>[] buckets = new ArrayList[bucketCount];
        for (int i = 0; i < bucketCount; i++) {
            buckets[i] = new ArrayList<>();
        }
        
        // 将元素放入桶中
        for (int item : arr) {
            int bucketIndex = (item - min) / arr.length;
            buckets[bucketIndex].add(item);
        }
        
        // 对每个桶内的元素排序
        for (List<Integer> bucket : buckets) {
            Collections.sort(bucket);
        }
        
        // 合并桶内元素
        int index = 0;
        for (List<Integer> bucket : buckets) {
            for (int item : bucket) {
                arr[index++] = item;
            }
        }
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {29, 25, 3, 49, 9, 37, 21, 43};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        bucketSort(arr);
        
        System.out.println("排序后数组：");
        printArray(arr);
    }
    
    // 打印数组
    public static void printArray(int[] arr) {
        for (int i : arr) {
            System.out.print(i + " ");
        }
        System.out.println();
    }
}
```

## 桶排序的优化

### 1. 桶的数量选择

桶的数量选择非常重要，它直接影响桶排序的效率：
- 桶太多：浪费空间，且桶合并操作变多
- 桶太少：每个桶内元素太多，桶内排序效率降低

一个常见的做法是让桶的数量与待排序数组的大小相同，即 n 个桶：

```java
int bucketCount = arr.length;
```

### 2. 动态调整桶的数量

根据数据分布动态调整桶的数量：

```java
int range = max - min;
int bucketCount = range / 10; // 每个桶跨度为10
if (bucketCount <= 1) bucketCount = 2;
if (bucketCount > arr.length / 2) bucketCount = arr.length / 2;
```

### 3. 均匀分布的桶映射

如果知道数据的分布情况，可以设计更合适的分桶函数，使各个桶中的元素数量尽可能均匀：

```java
// 对于均匀分布的数据
int bucketIndex = (item - min) * bucketCount / (max - min);

// 对于已知的区间数据
int bucketIndex = item / interval; // interval是预定义的桶间隔
```

## 算法复杂度分析

- **时间复杂度**：
  - 最好情况：O(n+k)，当输入数据均匀分布时（每个桶的元素数量为常数）
  - 最坏情况：O(n²)，当所有元素都映射到同一个桶中时
  - 平均情况：O(n)，当输入数据是均匀分布时

- **空间复杂度**：O(n+k)，需要额外的桶空间，其中n是数组大小，k是桶的数量。

- **稳定性**：稳定。如果桶内排序用的算法是稳定的（比如插入排序），那么桶排序也是稳定的。

## 适用场景

桶排序适用于：
1. 输入数据均匀分布在一个区间内
2. 区间范围不是特别大
3. 对内存空间要求不严格的场景

桶排序特别适合用于外部排序，即当待排序的数据量很大，无法一次性加载到内存中时，可以将数据分到多个较小的桶中，分别加载并排序。

## 桶排序的局限性

1. 如果数据分布不均匀，可能导致效率大幅下降
2. 需要额外的空间来存储桶
3. 浮点数的排序效果通常比整数好，因为浮点数更容易均匀分布

桶排序是一种分散式的排序算法，其效率高度依赖于输入数据的分布情况和映射函数的设计。在实际应用中，通常需要根据具体的数据特点来调整桶的数量和映射策略，以达到最佳性能。 