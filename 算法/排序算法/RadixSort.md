# 基数排序 (Radix Sort)

基数排序是一种非比较型整数排序算法，其原理是将整数按位数切割成不同的数字，然后按每个位数分别比较。基数排序的方式可以采用最低位优先（LSD Least Significant Digit）或最高位优先（MSD Most Significant Digit）。

## 基本思想

对于一组整数，基数排序的步骤如下：
1. 将所有待比较数值统一为同样的数位长度，数位较短的数前面补零
2. 从最低位（或最高位）开始，依次对每一位进行一次"分配"和"收集"
3. 分配：将所有元素按照当前位的值分配到对应的桶中（一般是0-9，共10个桶）
4. 收集：按照桶的顺序，将所有元素收集起来
5. 重复步骤2-4，直到所有位都完成排序

## 图解过程

![基数排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/radix-sort.gif)

## 代码实现

### 最低位优先（LSD）基数排序

```java
public class RadixSort {

    public static void radixSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        // 找出最大值
        int max = arr[0];
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] > max) {
                max = arr[i];
            }
        }
        
        // 计算最大值的位数
        int maxDigitLength = 0;
        while (max != 0) {
            max /= 10;
            maxDigitLength++;
        }
        
        // 初始化桶
        int mod = 10;
        int dev = 1;
        
        // 用于临时存储元素的数组
        int[][] counter = new int[10][arr.length];
        // 记录每个桶中存储元素的数量
        int[] count = new int[10];
        
        // 按照从右到左的顺序，对每一位进行排序
        for (int i = 0; i < maxDigitLength; i++, dev *= 10, mod *= 10) {
            // 分配：将元素分配到桶中
            for (int j = 0; j < arr.length; j++) {
                int digit = (arr[j] % mod) / dev;
                counter[digit][count[digit]++] = arr[j];
            }
            
            // 收集：将元素从桶中收集回数组
            int index = 0;
            for (int k = 0; k < 10; k++) {
                for (int j = 0; j < count[k]; j++) {
                    arr[index++] = counter[k][j];
                }
                // 重置计数器
                count[k] = 0;
            }
        }
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {170, 45, 75, 90, 802, 24, 2, 66};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        radixSort(arr);
        
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

### 使用计数排序作为基数排序的子过程

可以使用计数排序代替简单的桶排序作为基数排序中的子过程，这样可以减少空间使用：

```java
public static void radixSortUsingCounting(int[] arr) {
    if (arr == null || arr.length <= 1) {
        return;
    }
    
    // 找出最大值
    int max = arr[0];
    for (int i = 1; i < arr.length; i++) {
        if (arr[i] > max) {
            max = arr[i];
        }
    }
    
    // 计算最大值的位数
    int maxDigitLength = 0;
    while (max != 0) {
        max /= 10;
        maxDigitLength++;
    }
    
    // 临时数组，用于存储排序结果
    int[] output = new int[arr.length];
    // 基数（10进制）
    int radix = 10;
    
    // 按照从右到左的顺序，对每一位进行排序
    for (int position = 0, div = 1; position < maxDigitLength; position++, div *= radix) {
        // 计数数组，记录每个可能的数字（0-9）出现的次数
        int[] count = new int[radix];
        
        // 统计每个数字出现的次数
        for (int value : arr) {
            int digit = (value / div) % radix;
            count[digit]++;
        }
        
        // 将count[i]调整为位置索引，即小于等于i的元素个数
        for (int i = 1; i < radix; i++) {
            count[i] += count[i - 1];
        }
        
        // 从后向前遍历原始数组，确保排序的稳定性
        for (int i = arr.length - 1; i >= 0; i--) {
            int digit = (arr[i] / div) % radix;
            output[--count[digit]] = arr[i];
        }
        
        // 将排序结果复制回原始数组
        System.arraycopy(output, 0, arr, 0, arr.length);
    }
}
```

### 处理负数的基数排序

基本的基数排序无法直接处理负数。下面是一个可以处理负数的基数排序实现：

```java
public static void radixSortWithNegatives(int[] arr) {
    if (arr == null || arr.length <= 1) {
        return;
    }
    
    // 将数组分为负数和非负数两部分
    List<Integer> negatives = new ArrayList<>();
    List<Integer> nonNegatives = new ArrayList<>();
    
    for (int value : arr) {
        if (value < 0) {
            negatives.add(-value); // 取绝对值
        } else {
            nonNegatives.add(value);
        }
    }
    
    // 转换为数组
    int[] negArr = new int[negatives.size()];
    int[] nonNegArr = new int[nonNegatives.size()];
    
    for (int i = 0; i < negatives.size(); i++) {
        negArr[i] = negatives.get(i);
    }
    
    for (int i = 0; i < nonNegatives.size(); i++) {
        nonNegArr[i] = nonNegatives.get(i);
    }
    
    // 对两部分分别进行基数排序
    radixSort(negArr);
    radixSort(nonNegArr);
    
    // 将负数部分倒序放回原数组（取负后），再放入非负数部分
    int index = 0;
    for (int i = negArr.length - 1; i >= 0; i--) {
        arr[index++] = -negArr[i];
    }
    
    for (int value : nonNegArr) {
        arr[index++] = value;
    }
}
```

## 算法复杂度分析

- **时间复杂度**：O(n×k)，其中 n 是排序元素的个数，k 是数字位数。
  - 最好情况：O(n×k)
  - 最坏情况：O(n×k)
  - 平均情况：O(n×k)

- **空间复杂度**：O(n+k)，需要额外的存储空间。

- **稳定性**：稳定。基数排序是稳定的排序算法，相等的元素不会改变它们的相对位置。

## 适用场景

基数排序适用于：
1. 数据范围较小的整数排序
2. 整数位数不是特别大的情况
3. 需要稳定排序的场景
4. 数据量大但是每个数的位数不多的情况

基数排序在一些特定的应用中表现出色，例如：
- 字符串排序（例如字典序排序）
- 整数排序（例如学号、身份证号等）
- 日期时间排序

## 基数排序的优缺点

### 优点：
1. 可以避免进行比较操作
2. 在某些情况下效率高于比较排序
3. 是稳定的排序算法

### 缺点：
1. 空间复杂度较高
2. 只适用于整数或可转化为整数的数据
3. 当数据位数很多时，可能不如比较排序算法高效

## 基数排序与其他排序的比较

与计数排序和桶排序相比：
1. 基数排序可以处理较大范围的数据，而不像计数排序那样受限于数据范围
2. 基数排序不需要知道数据的分布情况，而桶排序效率高度依赖于数据分布
3. 基数排序的空间复杂度通常低于计数排序（当处理范围很大的数据时）

与比较排序（如快速排序、归并排序）相比：
1. 基数排序不需要数据之间的比较操作
2. 基数排序的性能不依赖于输入数据的初始顺序
3. 基数排序在处理某些特定类型的数据时可能比基于比较的排序算法更高效

基数排序是一种在特定场景下非常高效的排序算法，尤其适合处理整数或可以转化为整数的数据。 