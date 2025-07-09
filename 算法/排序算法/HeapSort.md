# 堆排序 (Heap Sort)

堆排序是一种基于比较的排序算法，它利用堆这种数据结构进行排序。堆是一个近似完全二叉树的结构，它满足堆的性质：父节点的值总是大于（或小于）其子节点的值。

## 基本思想

堆排序的基本思想是：
1. 将待排序序列构建成一个最大堆（或最小堆）
2. 将堆顶元素（最大值或最小值）与末尾元素交换
3. 调整剩余元素使其满足堆的性质，然后重复步骤2和3，直到整个序列有序

## 动图演示

![堆排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/heap-sort.gif)

## 堆的基本概念

在介绍堆排序之前，我们先了解下堆这个数据结构：

- **完全二叉树**：除了最后一层外，其他层的节点数都是满的，最后一层的节点都集中在左边
- **最大堆**：父节点的值大于或等于其子节点的值
- **最小堆**：父节点的值小于或等于其子节点的值

在堆排序中，通常使用数组来存储堆。对于数组中的位置为i的元素：
- 其父节点位置是 (i-1)/2（整除）
- 其左子节点位置是 2*i+1
- 其右子节点位置是 2*i+2

## 代码实现

以下是堆排序的Java实现，使用最大堆来进行升序排序：

```java
public class HeapSort {

    public static void heapSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        int n = arr.length;
        
        // 构建最大堆（从最后一个非叶节点开始）
        for (int i = n / 2 - 1; i >= 0; i--) {
            heapify(arr, n, i);
        }
        
        // 逐个将堆顶元素（最大值）移到数组末尾
        for (int i = n - 1; i > 0; i--) {
            // 将堆顶元素与当前未排序序列的末尾元素交换
            swap(arr, 0, i);
            
            // 重新调整堆
            heapify(arr, i, 0);
        }
    }
    
    /**
     * 调整堆，使其满足最大堆的性质
     * 
     * @param arr 待调整的数组
     * @param n 堆的大小
     * @param i 要调整的节点下标
     */
    private static void heapify(int[] arr, int n, int i) {
        int largest = i;    // 初始时，假设最大值是根节点
        int left = 2 * i + 1;   // 左子节点
        int right = 2 * i + 2;  // 右子节点
        
        // 如果左子节点大于根节点
        if (left < n && arr[left] > arr[largest]) {
            largest = left;
        }
        
        // 如果右子节点大于当前最大节点
        if (right < n && arr[right] > arr[largest]) {
            largest = right;
        }
        
        // 如果最大值不是根节点，则交换并继续调整
        if (largest != i) {
            swap(arr, i, largest);
            
            // 递归地调整被影响的子树
            heapify(arr, n, largest);
        }
    }
    
    private static void swap(int[] arr, int i, int j) {
        int temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {12, 11, 13, 5, 6, 7};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        heapSort(arr);
        
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

## 堆排序的性能优化

### 1. 迭代而非递归实现heapify

递归实现的heapify函数可能在极端情况下导致栈溢出。我们可以使用迭代实现：

```java
private static void heapifyIterative(int[] arr, int n, int i) {
    int largest = i;
    boolean isHeapified = false;
    
    while (!isHeapified) {
        int left = 2 * largest + 1;
        int right = 2 * largest + 2;
        int newLargest = largest;
        
        if (left < n && arr[left] > arr[newLargest]) {
            newLargest = left;
        }
        
        if (right < n && arr[right] > arr[newLargest]) {
            newLargest = right;
        }
        
        if (newLargest != largest) {
            swap(arr, largest, newLargest);
            largest = newLargest;
        } else {
            isHeapified = true;
        }
    }
}
```

### 2. 自底向上构建堆

自底向上构建堆可以减少heapify调用的次数：

```java
public static void heapSort(int[] arr) {
    if (arr == null || arr.length <= 1) {
        return;
    }
    
    int n = arr.length;
    
    // 自底向上构建堆
    for (int i = n / 2 - 1; i >= 0; i--) {
        heapify(arr, n, i);
    }
    
    // 逐个提取堆顶元素
    for (int i = n - 1; i > 0; i--) {
        swap(arr, 0, i);
        heapify(arr, i, 0);
    }
}
```

### 3. 仅当需要时才交换元素

在调整堆的过程中，可以采用"下沉"的策略，减少交换操作：

```java
private static void siftDown(int[] arr, int start, int end) {
    int root = start;
    int child;
    int toMove = arr[root];
    
    while ((child = 2 * root + 1) <= end) {
        if (child + 1 <= end && arr[child] < arr[child + 1]) {
            child++; // 右子节点更大
        }
        
        if (toMove >= arr[child]) {
            break;
        }
        
        arr[root] = arr[child];
        root = child;
    }
    
    arr[root] = toMove;
}
```

## 算法复杂度分析

- **时间复杂度**：
  - 构建初始堆：O(n)
  - 堆调整：O(logn)
  - 整个排序过程：O(nlogn)
  
  无论是最好、最坏还是平均情况，堆排序的时间复杂度都是O(nlogn)。

- **空间复杂度**：O(1)，仅使用常量级的额外空间。

- **稳定性**：不稳定。在交换堆顶元素和末尾元素的过程中，相等元素的相对位置可能发生改变。

## 堆排序的优缺点

### 优点：
1. 时间复杂度稳定，始终为O(nlogn)
2. 空间复杂度低，是原地排序算法
3. 适合处理大量数据的排序

### 缺点：
1. 实际运行效率通常比快速排序慢，常数因子较大
2. 不稳定的排序算法
3. 对缓存不友好，因为堆排序的操作可能跨越不同的内存区域

## 适用场景

堆排序适用于：
1. 需要稳定的O(nlogn)时间复杂度的场景
2. 内存资源有限的情况（因为是原地排序）
3. 需要找出数组中最大/最小的几个元素（使用堆来维护）

堆排序虽然不如快速排序常用，但在特定场景下仍然是重要的排序算法，尤其是在实现优先队列和处理TopK问题时。 