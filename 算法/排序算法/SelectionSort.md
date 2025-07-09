# 选择排序 (Selection Sort)

选择排序是一种简单直观的排序算法。它的工作原理是每次从待排序的数据中选出最小（或最大）的一个元素，存放在序列的起始位置，然后再从剩余未排序元素中继续寻找最小（或最大）元素，放到已排序序列的末尾。

## 基本思想

1. 在未排序序列中找到最小（大）元素，存放到排序序列的起始位置
2. 从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾
3. 重复第二步，直到所有元素均排序完毕

## 动图演示

![选择排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/selection-sort.gif)

## 代码实现

```java
public class SelectionSort {

    public static void selectionSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        int n = arr.length;
        
        // 遍历所有数组元素
        for (int i = 0; i < n - 1; i++) {
            // 找出最小元素索引
            int minIndex = i;
            for (int j = i + 1; j < n; j++) {
                if (arr[j] < arr[minIndex]) {
                    minIndex = j;
                }
            }
            
            // 将最小元素放到已排序序列的末尾
            if (minIndex != i) {
                int temp = arr[minIndex];
                arr[minIndex] = arr[i];
                arr[i] = temp;
            }
        }
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {64, 25, 12, 22, 11};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        selectionSort(arr);
        
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

## 双向选择排序优化

我们还可以实现双向选择排序，即每次同时找出最小值和最大值，这样可以减少循环次数：

```java
public static void bidirectionalSelectionSort(int[] arr) {
    if (arr == null || arr.length <= 1) {
        return;
    }
    
    int left = 0;
    int right = arr.length - 1;
    
    while (left < right) {
        int minIndex = left;
        int maxIndex = left;
        
        // 在当前未排序区间找出最小值和最大值
        for (int i = left + 1; i <= right; i++) {
            if (arr[i] < arr[minIndex]) {
                minIndex = i;
            }
            if (arr[i] > arr[maxIndex]) {
                maxIndex = i;
            }
        }
        
        // 将最小值交换到左边
        if (minIndex != left) {
            int temp = arr[left];
            arr[left] = arr[minIndex];
            arr[minIndex] = temp;
            
            // 如果最大值的位置是left，那么最大值现在在minIndex位置
            if (maxIndex == left) {
                maxIndex = minIndex;
            }
        }
        
        // 将最大值交换到右边
        if (maxIndex != right) {
            int temp = arr[right];
            arr[right] = arr[maxIndex];
            arr[maxIndex] = temp;
        }
        
        left++;
        right--;
    }
}
```

## 算法复杂度分析

- **时间复杂度**：
  - 最好情况：O(n²)
  - 最坏情况：O(n²)
  - 平均情况：O(n²)
  
  无论输入数据如何，时间复杂度都是一样的，因为无论什么情况都要进行n-1轮比较，每轮比较都需要扫描未排序区间的所有元素。

- **空间复杂度**：O(1)，只需要一个临时变量用于交换元素

- **稳定性**：不稳定。选择排序是给每个位置选择当前元素最小的，例如序列 5 8 5 2 9，第一次找到最小元素 2，与第一个位置的 5 交换，那么第一个 5 和中间的 5 顺序就变了，所以选择排序不是一个稳定的算法。

## 适用场景

选择排序适用于：
1. 数据规模较小的场景
2. 对算法的空间复杂度有较高要求的场景（因为它是原地排序算法）
3. 对数据交换操作成本较高的情况（选择排序交换次数少）

由于选择排序的时间复杂度较高，通常不适用于大规模数据的排序。但与冒泡排序相比，选择排序的交换操作更少，因此在某些特定场景下可能会有较好的性能。 