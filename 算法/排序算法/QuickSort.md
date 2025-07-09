# 快速排序 (Quick Sort)

快速排序是一种高效的、分治的排序算法，由C. A. R. Hoare于1960年提出。它的基本思想是通过一趟排序将待排记录分隔成独立的两部分，其中一部分记录的关键字均比另一部分的关键字小，然后分别对这两部分记录继续进行排序，以达到整个序列有序。

## 基本思想

快速排序使用分治法来把一个序列分为两个子序列：
1. 从数列中挑出一个元素，称为"基准"（pivot）
2. 重新排序数列，所有比基准值小的元素摆放在基准前面，所有比基准值大的元素摆在基准后面。在这个分区结束后，基准就处于数列的中间位置。这个操作称为分区（partition）操作
3. 递归地把小于基准值元素的子数列和大于基准值元素的子数列排序

## 动图演示

![快速排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/quick-sort.gif)

## 代码实现

### 基本实现

```java
public class QuickSort {

    public static void quickSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        quickSort(arr, 0, arr.length - 1);
    }
    
    private static void quickSort(int[] arr, int left, int right) {
        if (left < right) {
            // 找出基准元素，将数组分为两部分
            int pivotIndex = partition(arr, left, right);
            
            // 递归地对两部分进行排序
            quickSort(arr, left, pivotIndex - 1);
            quickSort(arr, pivotIndex + 1, right);
        }
    }
    
    private static int partition(int[] arr, int left, int right) {
        // 选择最右边的元素作为基准
        int pivot = arr[right];
        int i = left - 1; // 记录小于等于pivot的元素的位置
        
        for (int j = left; j < right; j++) {
            // 如果当前元素小于等于基准，交换它们
            if (arr[j] <= pivot) {
                i++;
                swap(arr, i, j);
            }
        }
        
        // 将基准放到正确的位置上
        swap(arr, i + 1, right);
        return i + 1;
    }
    
    private static void swap(int[] arr, int i, int j) {
        int temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {10, 7, 8, 9, 1, 5};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        quickSort(arr);
        
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

## 优化策略

### 1. 随机选择基准元素

最简单的快速排序实现通常选择第一个或最后一个元素作为基准。但在最坏情况下（数组已经有序或逆序），这会导致O(n²)的时间复杂度。通过随机选择基准元素，可以降低遇到最坏情况的可能性：

```java
private static int randomPartition(int[] arr, int left, int right) {
    int randomIndex = left + (int)(Math.random() * (right - left + 1));
    swap(arr, randomIndex, right);
    return partition(arr, left, right);
}

private static void quickSort(int[] arr, int left, int right) {
    if (left < right) {
        int pivotIndex = randomPartition(arr, left, right);
        quickSort(arr, left, pivotIndex - 1);
        quickSort(arr, pivotIndex + 1, right);
    }
}
```

### 2. 三数取中法选择基准

另一种常用的选择基准的方法是"三数取中"法，即取左端、中间、右端三个元素的中间值作为基准：

```java
private static int medianOfThree(int[] arr, int left, int mid, int right) {
    // 返回left、mid、right三个位置元素的中间值的索引
    if (arr[left] > arr[mid]) {
        if (arr[mid] > arr[right]) {
            return mid;
        } else if (arr[left] > arr[right]) {
            return right;
        } else {
            return left;
        }
    } else {
        if (arr[left] > arr[right]) {
            return left;
        } else if (arr[mid] > arr[right]) {
            return right;
        } else {
            return mid;
        }
    }
}

private static int partitionWithMedianOfThree(int[] arr, int left, int right) {
    int mid = left + (right - left) / 2;
    int medianIndex = medianOfThree(arr, left, mid, right);
    swap(arr, medianIndex, right);
    return partition(arr, left, right);
}

private static void quickSort(int[] arr, int left, int right) {
    if (left < right) {
        int pivotIndex = partitionWithMedianOfThree(arr, left, right);
        quickSort(arr, left, pivotIndex - 1);
        quickSort(arr, pivotIndex + 1, right);
    }
}
```

### 3. 对小数组使用插入排序

对于小规模数组（通常小于10个元素），插入排序可能比快速排序更高效：

```java
private static void quickSort(int[] arr, int left, int right) {
    // 当子数组长度小于等于10时，使用插入排序
    if (right - left <= 10) {
        insertionSort(arr, left, right);
        return;
    }
    
    if (left < right) {
        int pivotIndex = partition(arr, left, right);
        quickSort(arr, left, pivotIndex - 1);
        quickSort(arr, pivotIndex + 1, right);
    }
}

private static void insertionSort(int[] arr, int left, int right) {
    for (int i = left + 1; i <= right; i++) {
        int key = arr[i];
        int j = i - 1;
        while (j >= left && arr[j] > key) {
            arr[j + 1] = arr[j];
            j--;
        }
        arr[j + 1] = key;
    }
}
```

### 4. 双枢轴快速排序（Dual-Pivot QuickSort）

这是一种更先进的快速排序变体，使用两个枢轴点而不是一个：

```java
public static void dualPivotQuickSort(int[] arr, int left, int right) {
    if (right - left >= 1) {
        if (arr[left] > arr[right]) {
            swap(arr, left, right);
        }
        
        int p = arr[left];     // 第一个枢轴点
        int q = arr[right];    // 第二个枢轴点
        
        int l = left + 1;      // 指向小于p区域的右边界
        int g = right - 1;     // 指向大于q区域的左边界
        int k = l;             // 当前扫描的元素
        
        while (k <= g) {
            if (arr[k] < p) {
                swap(arr, k, l);
                l++;
            } else if (arr[k] >= q) {
                while (arr[g] > q && k < g) {
                    g--;
                }
                swap(arr, k, g);
                g--;
                if (arr[k] < p) {
                    swap(arr, k, l);
                    l++;
                }
            }
            k++;
        }
        
        l--;
        g++;
        
        swap(arr, left, l);
        swap(arr, right, g);
        
        dualPivotQuickSort(arr, left, l - 1);
        dualPivotQuickSort(arr, l + 1, g - 1);
        dualPivotQuickSort(arr, g + 1, right);
    }
}
```

Java中的Arrays.sort()方法使用的就是双枢轴快速排序算法。

## 算法复杂度分析

- **时间复杂度**：
  - 最好情况：O(nlogn)，当每次划分都能将数组均匀地分为两部分时
  - 最坏情况：O(n²)，当每次划分只能减少一个元素时（例如数组已经有序）
  - 平均情况：O(nlogn)
  
- **空间复杂度**：O(logn)，主要是递归调用栈的空间。在最坏情况下（即数组已经有序或逆序时），递归深度可达O(n)，因此空间复杂度为O(n)。

- **稳定性**：不稳定。快速排序可能会改变相同元素的相对位置。

## 适用场景

快速排序适用于：
1. 大规模数据的内部排序
2. 平均情况下需要高效排序的场景
3. 不需要排序稳定性的场景
4. 内存空间有限的场景（相比归并排序）

快速排序是实际应用中最常用的排序算法之一，因为它在平均情况下的性能非常好，常数因子较小，并且可以在原地进行排序。但在特定情况下（如处理已经排序好的数据），如果不加优化，性能可能会急剧下降。 