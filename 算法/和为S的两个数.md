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
可以进一步改进的是进行了很多没有必要的计算，如【2，4，5，6，7，8】这个数组，target为8，2找到6发现大了之后停止，4就不需要考虑6之后所有的数了，因为一定比target大，所以从5开始往前找，找到小的停止。这就已经有双指针的意思了，2也不需要从前往后找，直接从后往前找，如果小了就停，往前走一步，重复这个步骤。由此经过几步优化从暴力解成为O(N2)的解。优化的关键有：和相同的两个数，差大的乘积更小，所以从头开始找，找到了就可以返回；不进行重复计算。
```java
public class Solution {
    public ArrayList<Integer>  FindNumbersWithSum(int [] array,int sum) {
        int l = 0;
        int r = array.length - 1;
        while (l < r) {
            int s = array[l] + array[r];
            if (s == sum) {
                return new ArrayList<Integer>(Arrays.asList(array[l], array[r]));
            } else if (s > sum) {
                r--;
            } else {
                l++;
            }
        }
        return new ArrayList<Integer>();
    }
```
