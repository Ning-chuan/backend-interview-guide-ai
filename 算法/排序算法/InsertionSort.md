# 插入排序 (Insertion Sort)

插入排序是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。

## 基本思想

1. 将第一个元素视为已排序序列，其余元素视为未排序序列
2. 从头到尾遍历未排序序列，将扫描到的每个元素插入到已排序序列的适当位置（如果待插入的元素与有序序列中的某个元素相等，则将待插入元素插入到相等元素的后面）
3. 重复步骤2，直到未排序序列元素全部插入到已排序序列中

## 动图演示

![插入排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/insertion-sort.gif)

## 代码实现

```java
public class InsertionSort {

    public static void insertionSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        int n = arr.length;
        
        // 从第二个元素开始遍历，第一个元素被视为已排序序列
        for (int i = 1; i < n; i++) {
            // 记录待插入元素
            int key = arr[i];
            int j = i - 1;
            
            // 将比key大的元素向右移动
            while (j >= 0 && arr[j] > key) {
                arr[j + 1] = arr[j];
                j--;
            }
            
            // 插入key到适当位置
            arr[j + 1] = key;
        }
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {12, 11, 13, 5, 6};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        insertionSort(arr);
        
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

## 二分查找优化

我们可以使用二分查找来优化插入排序。在插入元素时，不再逐个比较，而是使用二分查找快速找到插入位置：

```java
public static void binaryInsertionSort(int[] arr) {
    if (arr == null || arr.length <= 1) {
        return;
    }
    
    int n = arr.length;
    
    for (int i = 1; i < n; i++) {
        int key = arr[i];
        int left = 0;
        int right = i - 1;
        
        // 二分查找，找到插入位置
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (arr[mid] > key) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        
        // left 就是要插入的位置
        // 将 [left, i-1] 区间的元素向右移动一位
        for (int j = i - 1; j >= left; j--) {
            arr[j + 1] = arr[j];
        }
        
        arr[left] = key;
    }
}
```

虽然二分查找可以减少比较次数，但元素移动的操作仍然是O(n²)，所以整体时间复杂度仍为O(n²)。但在比较操作非常耗时的情况下，这种优化是有意义的。

## 算法复杂度分析

- **时间复杂度**：
  - 最好情况：O(n)，当输入数组已经排序时
  - 最坏情况：O(n²)，当输入数组是逆序时
  - 平均情况：O(n²)

- **空间复杂度**：O(1)，只需要一个临时变量key

- **稳定性**：稳定。插入排序是稳定的排序算法，相等元素的相对位置在排序后不会改变。

## 适用场景

插入排序适用于：
1. 数据量小的排序任务
2. 数据已经基本有序的情况（最好情况下可达到O(n)的时间复杂度）
3. 对稳定性有要求的场景
4. 作为其他复杂排序算法的子过程

实际应用中，当数据规模小于一定阈值时，很多语言的标准库排序函数会使用插入排序而非快速排序，因为对于小数据集，插入排序通常比快速排序更高效。例如，在Java中，Arrays.sort()对于小数组（通常小于47个元素）会使用插入排序。 