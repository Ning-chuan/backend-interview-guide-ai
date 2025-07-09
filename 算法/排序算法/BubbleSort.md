# 冒泡排序 (Bubble Sort)

冒泡排序是一种简单直观的排序算法。它重复地遍历要排序的数列，一次比较两个元素，如果它们的顺序错误就把它们交换过来。遍历数列的工作是重复地进行直到没有再需要交换的元素，也就是说该数列已经排序完成。

## 基本思想

1. 比较相邻的元素。如果第一个比第二个大，就交换它们两个；
2. 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对，这样在最后的元素应该会是最大的数；
3. 针对所有的元素重复以上的步骤，除了最后一个；
4. 重复上述步骤，直到没有任何一对数字需要比较。

## 动图演示

![冒泡排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/bubble-sort.gif)

## 代码实现

```java
public class BubbleSort {
    
    public static void bubbleSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        int n = arr.length;
        for (int i = 0; i < n - 1; i++) {
            boolean swapped = false;
            
            // 每轮比较相邻元素
            for (int j = 0; j < n - 1 - i; j++) {
                if (arr[j] > arr[j + 1]) {
                    // 交换arr[j]和arr[j+1]
                    int temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                    swapped = true;
                }
            }
            
            // 如果没有发生交换，则数组已经有序
            if (!swapped) {
                break;
            }
        }
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {64, 34, 25, 12, 22, 11, 90};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        bubbleSort(arr);
        
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

## 优化版本

上述代码中已经包含了一个优化：当某一轮遍历中没有发生交换时，说明数组已经有序，可以提前结束排序。还可以进一步优化：

```java
public static void optimizedBubbleSort(int[] arr) {
    if (arr == null || arr.length <= 1) {
        return;
    }
    
    int n = arr.length;
    // 记录最后一次交换的位置
    int lastSwappedIndex = 0;
    // 无序数列的边界，每次比较只需要比到这里为止
    int sortBorder = n - 1;
    
    for (int i = 0; i < n - 1; i++) {
        boolean swapped = false;
        
        for (int j = 0; j < sortBorder; j++) {
            if (arr[j] > arr[j + 1]) {
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
                swapped = true;
                lastSwappedIndex = j;
            }
        }
        sortBorder = lastSwappedIndex;
        
        if (!swapped) {
            break;
        }
    }
}
```

## 算法复杂度分析

- **时间复杂度**：
  - 最好情况：O(n)，当输入数组已经排序时
  - 最坏情况：O(n²)，当输入数组是逆序时
  - 平均情况：O(n²)
  
- **空间复杂度**：O(1)，只需要一个临时变量用于交换元素

- **稳定性**：稳定。相等的元素不会交换位置，所以它们的相对位置保持不变。

## 适用场景

冒泡排序由于其简单性和稳定性，适用于：
1. 小规模数据的排序
2. 对稳定性有要求的场景
3. 算法教学或入门学习排序算法

由于其平均时间复杂度较高，不适合大规模数据的排序。 