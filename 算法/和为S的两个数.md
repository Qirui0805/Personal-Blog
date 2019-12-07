#### 题目描述
输入一个递增排序的数组和一个数字S，在数组中查找两个数，使得他们的和正好是S，如果有多对数字的和等于S，输出两个数的乘积最小的。
输出描述:
对应每个测试案例，输出两个数，小的先输出。
#### 题目类型
双指针
#### 解题思路
- 暴力解 O(N2)
两层循环，找到第一个符合条件的后就可以返回了，和相等的两个数，差越大，乘积越小
```java
public class Solution{
  public ArrayList<Integer> FindNumbersWithSum(int [] array,int sum) {
    if (array.length == 0)
      return new ArrayList<Integer>();
    for (int i = 0; i < array.length; i++) {
      for (int j = i + 1; j < array.length; j++) {
        if (array[i] + array[j] == sum)
          return new Arrays.asList(array[i], array[j]);
      }
    }
    return new ArrayList<Integer>();
  }
}
``` 
当然也可以把循环完整走一遍，再把符合条件的乘积都算出来，不过这就是暴力中的暴力了。
第一点可以改进的地方是当和大于target时就可以停止
```java
public class Solution{
  public ArrayList<Integer> FindNumbersWithSum(int [] array,int sum) {
    if (array.length == 0)
      return new ArrayList<Integer>();
    for (int i = 0; i < array.length; i++) {
      for (int j = i + 1; j < array.length && array[i] + array[j] <= sum; j++) {
        if (array[i] + array[j] == sum)
          return new Arrays.asList(array[i], array[j]);
      }
    }
    return new ArrayList<Integer>();
  }
}
``` 
- 双指针
可以进一步改进的原因是进行很多没有必要的计算，如【2，4，6，7，8】这个数组，target为8，2找到6发现大了之后停止，4就不需要考虑6之后
