# 希尔排序 (Shell Sort)

希尔排序是插入排序的一种更高效的改进版本，它是第一个突破O(n²)的排序算法。希尔排序是非稳定排序算法。

## 基本思想

1. 先将整个待排序的序列分割成若干子序列（按照一定的增量分组）分别进行直接插入排序
2. 待整个序列基本有序时，再对全体记录进行一次直接插入排序

希尔排序的核心在于间隔序列（增量序列）的设定，不同的间隔序列会导致不同的时间复杂度。

## 算法步骤

1. 选择一个增量序列t1, t2, ..., tk，其中ti > tj（当i < j），tk = 1
2. 按增量序列个数k，对序列进行k趟排序
3. 每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m的子序列，分别对各子表进行直接插入排序，仅增量因子为1时，整个序列作为一个表来处理，表长度即为整个序列的长度

## 动图演示

![希尔排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/shell-sort.gif)

## 代码实现

以下是希尔排序的Java实现，使用常见的增量序列gap = gap / 2：

```java
public class ShellSort {

    public static void shellSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        int n = arr.length;
        
        // 初始增量为数组长度的一半，每次减半
        for (int gap = n / 2; gap > 0; gap /= 2) {
            // 对各个分组进行插入排序，每个分组中的元素在原始数组中相隔gap个元素
            for (int i = gap; i < n; i++) {
                // 对arr[i]进行插入排序
                int temp = arr[i];
                int j = i;
                
                // 将较大的元素向右移动
                while (j >= gap && arr[j - gap] > temp) {
                    arr[j] = arr[j - gap];
                    j -= gap;
                }
                
                // 插入元素
                arr[j] = temp;
            }
        }
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {12, 34, 54, 2, 3, 23, 19, 98, 9, 4};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        shellSort(arr);
        
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

## 其他增量序列

除了上述代码中使用的增量序列（每次除以2），还有其他几种常见的增量序列：

### 希尔增量序列

这是最原始的增量序列，即n/2, n/4, ..., 1，就是上面代码中使用的序列。

### Hibbard增量序列

Hibbard增量序列：1, 3, 7, ..., 2^k-1

```java
public static void shellSortHibbard(int[] arr) {
    int n = arr.length;
    
    // 计算最大的增量
    int h = 1;
    while (h <= n / 3) {
        h = h * 2 + 1; // 生成2^k-1序列：1, 3, 7, 15, 31, ...
    }
    
    // 使用Hibbard增量进行排序
    while (h >= 1) {
        for (int i = h; i < n; i++) {
            // 将arr[i]插入到arr[i-h], arr[i-2*h], arr[i-3*h]... 中
            int temp = arr[i];
            int j = i;
            
            while (j >= h && arr[j - h] > temp) {
                arr[j] = arr[j - h];
                j -= h;
            }
            
            arr[j] = temp;
        }
        h = (h - 1) / 2; // 缩小增量
    }
}
```

### Sedgewick增量序列

Sedgewick增量序列：1, 5, 19, 41, 109, ...，该序列是由 9×4^i - 9×2^i + 1 和 4^i - 3×2^i + 1 交替生成。

```java
public static void shellSortSedgewick(int[] arr) {
    int n = arr.length;
    
    // Sedgewick增量序列：1, 5, 19, 41, 109, ...
    int[] gaps = {1, 5, 19, 41, 109, 209, 505, 929, 2161, 3905, 8929, 16001, 36289, 
                  64769, 146305, 260609};
    
    // 找到适合的初始增量
    int k = 0;
    while (k < gaps.length && gaps[k] < n / 3) {
        k++;
    }
    
    // 使用Sedgewick增量进行排序
    while (k >= 0) {
        int gap = gaps[k--];
        
        for (int i = gap; i < n; i++) {
            int temp = arr[i];
            int j = i;
            
            while (j >= gap && arr[j - gap] > temp) {
                arr[j] = arr[j - gap];
                j -= gap;
            }
            
            arr[j] = temp;
        }
    }
}
```

## 算法复杂度分析

- **时间复杂度**：
  - 最好情况：根据增量序列的不同而不同
  - 最坏情况：O(n²)（使用希尔增量时）
  - 平均情况：O(nlog²n)（使用希尔增量时），使用Hibbard增量时为O(n^(3/2))，使用Sedgewick增量时接近O(n^(4/3))
  
- **空间复杂度**：O(1)，只需要一个临时变量

- **稳定性**：不稳定。由于相同的元素可能在不同的分组中排序，可能会改变它们的相对位置，所以希尔排序是不稳定的。

## 适用场景

希尔排序适用于：
1. 数据量适中（几千到几万）的排序任务
2. 不要求排序稳定性的场景
3. 中等大小的数组，对于此类情况，希尔排序可能比更复杂的O(nlogn)排序算法更快

希尔排序是一种实用的排序算法，它在中等大小的数组上表现良好，并且实现简单。然而，对于非常大的数据集，通常会选择归并排序、快速排序或堆排序等算法。 