# 计数排序 (Counting Sort)

计数排序是一种非比较型整数排序算法。它的核心思想是统计数组中每个值的出现次数，然后按顺序把数组中的值填充回原数组。计数排序要求输入的数据必须是有确定范围的整数。

## 基本思想

1. 找出待排序数组中的最大值和最小值
2. 统计数组中每个值为i的元素出现的次数，存入计数数组C的第i项
3. 对所有的计数累加（从C中的第一个元素开始，每一项和前一项相加）
4. 反向填充目标数组：将每个元素i放在新数组的第C(i)项，每放一个元素就将C(i)减1

## 图解过程

![计数排序](https://picgo-1258602555.cos.ap-nanjing.myqcloud.com/counting-sort.gif)

## 代码实现

### 基本实现

```java
public class CountingSort {

    public static void countingSort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }
        
        // 找出数组中的最大值和最小值
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
        
        // 计算计数数组的大小
        int range = max - min + 1;
        
        // 创建计数数组并统计每个元素的出现次数
        int[] count = new int[range];
        for (int i = 0; i < arr.length; i++) {
            count[arr[i] - min]++;
        }
        
        // 根据计数数组重构原始数组
        int index = 0;
        for (int i = 0; i < range; i++) {
            while (count[i] > 0) {
                arr[index++] = i + min;
                count[i]--;
            }
        }
    }
    
    // 测试
    public static void main(String[] args) {
        int[] arr = {8, 3, 5, 2, 9, 1, 4, 7, 6, 8, 3, 5, 8};
        
        System.out.println("排序前数组：");
        printArray(arr);
        
        countingSort(arr);
        
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

### 稳定的计数排序实现

上面的实现是不稳定的，以下是稳定版本的计数排序：

```java
public static void stableCountingSort(int[] arr) {
    if (arr == null || arr.length <= 1) {
        return;
    }
    
    // 找出数组中的最大值和最小值
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
    
    // 计算计数数组的大小
    int range = max - min + 1;
    
    // 创建计数数组并统计每个元素的出现次数
    int[] count = new int[range];
    for (int i = 0; i < arr.length; i++) {
        count[arr[i] - min]++;
    }
    
    // 计数数组累加，count[i]表示小于等于i+min的元素个数
    for (int i = 1; i < range; i++) {
        count[i] += count[i - 1];
    }
    
    // 创建临时数组，用于存储排序后的结果
    int[] output = new int[arr.length];
    
    // 从后向前扫描原始数组，确保稳定性
    for (int i = arr.length - 1; i >= 0; i--) {
        output[count[arr[i] - min] - 1] = arr[i];
        count[arr[i] - min]--;
    }
    
    // 将结果复制回原始数组
    System.arraycopy(output, 0, arr, 0, arr.length);
}
```

## 对象数组的计数排序

如果需要对包含多个属性的对象数组进行排序，可以这样实现：

```java
public static void objectCountingSort(Person[] persons) {
    if (persons == null || persons.length <= 1) {
        return;
    }
    
    // 找出数组中的最大年龄和最小年龄
    int maxAge = persons[0].getAge();
    int minAge = persons[0].getAge();
    for (int i = 1; i < persons.length; i++) {
        if (persons[i].getAge() > maxAge) {
            maxAge = persons[i].getAge();
        }
        if (persons[i].getAge() < minAge) {
            minAge = persons[i].getAge();
        }
    }
    
    // 计算计数数组的大小
    int range = maxAge - minAge + 1;
    
    // 创建计数数组并统计每个年龄的出现次数
    int[] count = new int[range];
    for (Person p : persons) {
        count[p.getAge() - minAge]++;
    }
    
    // 计数数组累加
    for (int i = 1; i < range; i++) {
        count[i] += count[i - 1];
    }
    
    // 创建临时数组，用于存储排序后的结果
    Person[] output = new Person[persons.length];
    
    // 从后向前扫描原始数组，确保稳定性
    for (int i = persons.length - 1; i >= 0; i--) {
        output[count[persons[i].getAge() - minAge] - 1] = persons[i];
        count[persons[i].getAge() - minAge]--;
    }
    
    // 将结果复制回原始数组
    System.arraycopy(output, 0, persons, 0, persons.length);
}

// Person类示例
class Person {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public int getAge() {
        return age;
    }
    
    public String getName() {
        return name;
    }
    
    @Override
    public String toString() {
        return name + "(" + age + ")";
    }
}
```

## 算法复杂度分析

- **时间复杂度**：O(n+k)，其中n是数组长度，k是整数的范围（即最大值与最小值的差加1）。
  - 最好情况：O(n+k)
  - 最坏情况：O(n+k)
  - 平均情况：O(n+k)

- **空间复杂度**：O(n+k)，需要额外的计数数组和输出数组。其中n是原始数组的大小，k是整数的范围。

- **稳定性**：稳定（使用稳定版本的实现）。在从后向前遍历并填充输出数组的过程中，相同值的元素会保持它们原有的相对顺序。

## 适用场景

计数排序适用于：
1. 元素是非负整数（或者可以转换为非负整数）
2. 元素的范围不是特别大（k不是特别大）
3. 需要排序的数据量很大，但元素取值范围相对较小

计数排序的优势在于当k较小时，它可以在线性时间内完成排序，这比基于比较的排序算法（如快速排序、归并排序等）的O(nlogn)的下界还要快。

## 计数排序的局限性

1. 计数排序要求输入的数据必须是有确定范围的整数
2. 如果k很大（例如，排序的元素是10亿以内的整数），那么计数排序的空间消耗将变得非常大
3. 不适合对浮点数等非整数数据进行排序

计数排序是一种用空间换时间的排序算法，在特定情况下效率非常高，但它也有明显的适用范围限制。 