# 归并排序 (Merge Sort)

归并排序是一种高效的、基于分治思想的排序算法。它将待排序的数组分成两个子数组，分别对它们进行排序，然后将两个已排序的子数组合并为一个。

## 基本思想

归并排序的核心思想是分治法（Divide and Conquer）：
1. 分解：将原问题分解为若干个规模较小的子问题
2. 解决：递归地求解各个子问题
3. 合并：将子问题的解合并成原问题的解

## 算法步骤

1. 将长度为n的输入序列分成两个长度为n/2的子序列
2. 对这两个子序列分别采用归并排序
3. 将两个排序好的子序列合并成一个最终的排序序列

## 动图演示

![归并排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/merge-sort.gif)

## 代码实现

```java
public class MergeSort {

    public static void mergeSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        // 创建一个与原数组等长的临时数组
        int[] temp = new int[arr.length];
        mergeSort(arr, 0, arr.length - 1, temp);
    }
    
    // 对arr[left...right]进行排序
    private static void mergeSort(int[] arr, int left, int right, int[] temp) {
        if (left < right) {
            int mid = left + (right - left) / 2;
            
            // 分别对左右两部分进行排序
            mergeSort(arr, left, mid, temp);
            mergeSort(arr, mid + 1, right, temp);
            
            // 合并两个已排序的子数组
            merge(arr, left, mid, right, temp);
        }
    }
    
    // 合并arr[left...mid]和arr[mid+1...right]两个有序数组
    private static void merge(int[] arr, int left, int mid, int right, int[] temp) {
        // 初始化i, j为左右两个子数组的起始下标
        int i = left;
        int j = mid + 1;
        int t = 0;  // 临时数组的起始下标
        
        // 比较左右两部分的元素，将较小的元素放入临时数组
        while (i <= mid && j <= right) {
            if (arr[i] <= arr[j]) {
                temp[t++] = arr[i++];
            } else {
                temp[t++] = arr[j++];
            }
        }
        
        // 如果左半部分还有剩余元素，将其全部放入临时数组
        while (i <= mid) {
            temp[t++] = arr[i++];
        }
        
        // 如果右半部分还有剩余元素，将其全部放入临时数组
        while (j <= right) {
            temp[t++] = arr[j++];
        }
        
        // 将临时数组中的元素复制回原数组
        t = 0;
        while (left <= right) {
            arr[left++] = temp[t++];
        }
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {38, 27, 43, 3, 9, 82, 10};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        mergeSort(arr);
        
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

## 自底向上实现归并排序

除了上述的递归实现，归并排序也可以通过自底向上的方式实现，避免递归调用：

```java
public static void mergeSortBottomUp(int[] arr) {
    if (arr == null || arr.length <= 1) {
        return;
    }
    
    int n = arr.length;
    int[] temp = new int[n];
    
    // size表示子数组的大小，从1开始，每次翻倍
    for (int size = 1; size < n; size *= 2) {
        // 对大小为size的相邻子数组进行合并
        for (int left = 0; left < n - size; left += 2 * size) {
            int mid = left + size - 1;
            int right = Math.min(left + 2 * size - 1, n - 1);
            merge(arr, left, mid, right, temp);
        }
    }
}

// 合并函数与递归版本相同
```

## 算法复杂度分析

- **时间复杂度**：
  - 最好情况：O(nlogn)
  - 最坏情况：O(nlogn)
  - 平均情况：O(nlogn)
  
  无论输入数据的分布如何，归并排序的时间复杂度都是O(nlogn)，这是它的一个主要优点。

- **空间复杂度**：O(n)，需要一个与原数组同样大小的临时数组。

- **稳定性**：稳定。归并排序会保持相等元素的相对顺序，因此它是稳定的排序算法。

## 优化策略

1. **小数组使用插入排序**：对于小规模子数组（例如长度小于一定阈值），使用插入排序代替递归的归并排序。
   
   ```java
   private static void mergeSort(int[] arr, int left, int right, int[] temp) {
       // 当子数组长度小于等于15时，使用插入排序
       if (right - left <= 15) {
           insertionSort(arr, left, right);
           return;
       }
       
       int mid = left + (right - left) / 2;
       mergeSort(arr, left, mid, temp);
       mergeSort(arr, mid + 1, right, temp);
       merge(arr, left, mid, right, temp);
   }
   ```

2. **检查是否需要合并**：如果arr[mid] <= arr[mid+1]，说明两个子数组已经有序且不需要合并。

   ```java
   private static void mergeSort(int[] arr, int left, int right, int[] temp) {
       if (left < right) {
           int mid = left + (right - left) / 2;
           mergeSort(arr, left, mid, temp);
           mergeSort(arr, mid + 1, right, temp);
           
           // 如果已经有序，则跳过合并步骤
           if (arr[mid] <= arr[mid + 1]) {
               return;
           }
           
           merge(arr, left, mid, right, temp);
       }
   }
   ```

## 适用场景

归并排序适用于：
1. 要求排序稳定的场景
2. 数据量较大且对时间复杂度要求较高的情况
3. 外部排序（当待排序数据不能全部加载到内存中时）
4. 链表排序（对链表进行归并排序不需要额外的空间）

归并排序的主要缺点是需要O(n)的额外空间，这在某些内存受限的环境中可能是一个问题。但它的稳定性和可预测的性能使其成为许多实际应用中的重要排序算法。 