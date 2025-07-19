---
title: LeetCode1-两数之和
tags:
  - 算法
  - LeetCode
  - 哈希表
categories:
  - 算法
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250719145604416.png'
toc: true
abbrlink: e5d20eb8
date: 2025-07-19 14:10:47
---

## 题目

给定一个整数数组 `nums` 和一个整数目标值 `target`，请你在该数组中找出和为目标值 `target` 的那两个整数，并返回它们的数组下标。
你可以假设每种输入只会对应一个答案，并且你不能使用两次相同的元素。
你可以按任意顺序返回答案。

**示例 1：**
- 输入：`nums = [2,7,11,15]`, `target = 9`
- 输出：`[0,1]`
- 解释：因为 `nums[0] + nums[1] == 9` ，返回 `[0, 1]` 。

**示例 2：**
- 输入：`nums = [3,2,4]`, `target = 6`
- 输出：`[1,2]`

**示例 3：**
- 输入：`nums = [3,3]`, `target = 6`
- 输出：`[0,1]`


**提示：**
- `2 <= nums.length <= 104`
- `-109 <= nums[i] <= 109`
- `-109 <= target <= 109`
- 只会存在一个有效答案

## 分析

这道题的目标非常明确：在一个整数数组中，找到两个数，使它们的和等于一个给定的目标值 `target`，然后返回这两个数的索引。

我们可以从最直观的思路开始分析：

1.  **暴力解法（Brute Force）**

    * 这是最容易想到的方法。我们可以遍历数组中的每一个元素 `x`，然后再遍历数组中剩下的元素 `y`，判断 `x + y` 是否等于 `target`。
    * 为了实现这个思路，我们可以使用两层循环。第一层循环选择第一个数，第二层循环选择第二个数。
    * **时间复杂度**：假设数组的长度是 `n`。外层循环需要执行 `n` 次，内层循环平均需要执行 `n/2` 次，所以总的时间复杂度是 $O(n^2)$。
    * **空间复杂度**：我们没有使用额外的存储空间（除了存放结果的数组），所以空间复杂度是 $O(1)$。
    * 对于这道题的`提示`中提到的 `nums.length <= 10^4`，`n^2` 意味着运算次数可能达到 `10^8`，这在很多在线判题系统中可能会超时。因此，我们需要寻找更优的解法。

2.  **优化思路：空间换时间**

    * 暴力解法的瓶颈在于，对于每个数 `x`，我们都需要通过再次遍历数组来寻找是否存在一个数 `y` 等于 `target - x`。这个“寻找”的过程太慢了。
    * 我们能否加速这个“寻找”过程呢？答案是肯定的。我们可以使用一种数据结构，它能够提供快速的查找功能。哈希表（Hash Map）就是理想的选择。
    * 哈希表可以在平均 $O(1)$ 的时间复杂度内完成插入和查找操作。

### 解题思路 (哈希表法)

我们可以利用哈希表来将查找目标配对数的时间复杂度从 $O(n)$ 降低到 $O(1)$。

具体的思路如下：

1.  创建一个哈希表（在 Java 中是 `HashMap`），用于存储数组中的元素及其对应的索引。键（Key）是数组的元素值，值（Value）是该元素的索引。

2.  遍历整个 `nums` 数组，对于数组中的每一个元素 `nums[i]`：
    a.  计算我们需要寻找的“配对数”，即 `complement = target - nums[i]`。
    b.  检查哈希表中是否存在键为 `complement` 的元素。
    \* **如果存在**：说明我们找到了配对的两个数。`nums[i]` 是其中一个，而 `complement` 是另一个。由于哈希表中已经存储了 `complement` 的索引，我们就可以直接返回这两个数的索引：`[map.get(complement), i]`。
    \* **如果不存在**：说明到目前为止，还没有找到能够与 `nums[i]` 配对的数。为了让后面的元素能找到 `nums[i]` 进行配对，我们将当前的元素值 `nums[i]` 和它的索引 `i` 存入哈希表中。

3.  由于题目保证“只会存在一个有效答案”，所以我们一定能在遍历结束前找到答案并返回。

**为什么这个方法是有效的？**

当我们遍历到 `nums[i]` 时，我们实际上是在问：“数组中前面（已经被遍历过的元素）是否存在一个数，它等于 `target - nums[i]`？”。哈希表 `map` 就扮演了这个“记录员”的角色，它高效地存储了所有我们已经“见过”的数字及其索引。通过查询哈希表，我们就能以 $O(1)$ 的时间复杂度回答这个问题。

整个算法只需要遍历一次数组，每次遍历时都进行一次哈希表的插入和查找操作（均为 $O(1)$），所以总的时间复杂度是 $O(n)$，空间复杂度是 $O(n)$（因为最坏情况下，我们需要将所有元素都存入哈希表）。相比暴力解法，这是一个巨大的提升。

### 总结

上面的 Java 代码通过一次遍历和哈希表的使用，高效地解决了“两数之和”问题。

* **核心数据结构**：`java.util.HashMap`
* **算法思想**：以空间换时间。在遍历数组的同时，使用哈希表记录已经访问过的数字及其索引。对于每个新数字，我们不再需要回头遍历整个数组去寻找配对，而是直接在哈希表中以 $O(1)$ 的时间复杂度查找是否存在能与之相加得到目标值的“补数”。
* **时间复杂度**：$O(n)$，因为数组只被遍历了一次。
* **空间复杂度**：$O(n)$，因为在最坏的情况下，哈希表可能需要存储数组中所有的 `n` 个元素。

## 答案

```java
import java.util.HashMap;
import java.util.Map;

class Solution {
    /**
     * 给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target 的那 两个 整数，并返回它们的数组下标。
     *
     * @param nums   整数数组
     * @param target 整数目标值
     * @return 包含两个整数下标的数组
     */
    public int[] twoSum(int[] nums, int target) {
        // 创建一个 HashMap，键存储数组中的数字，值存储该数字的索引
        Map<Integer, Integer> map = new HashMap<>();

        // 遍历数组
        for (int i = 0; i < nums.length; i++) {
            // 计算需要找到的另一个数（补数）
            int complement = target - nums[i];

            // 检查 HashMap 中是否存在这个补数
            if (map.containsKey(complement)) {
                // 如果存在，说明找到了答案
                // map.get(complement) 返回的是补数的索引
                // i 是当前数的索引
                return new int[]{map.get(complement), i};
            }

            // 如果不存在，将当前的数和它的索引存入 HashMap
            // 供后续的元素进行匹配
            map.put(nums[i], i);
        }

        // 根据题目提示“只会存在一个有效答案”，程序不会执行到这里
        // 但为了代码的完整性，可以抛出一个异常
        throw new IllegalArgumentException("No two sum solution");
    }
}

// 示例使用
public class Main {
    public static void main(String[] args) {
        Solution solution = new Solution();

        // 示例 1
        int[] nums1 = {2, 7, 11, 15};
        int target1 = 9;
        int[] result1 = solution.twoSum(nums1, target1);
        System.out.println("示例 1 输出: [" + result1[0] + "," + result1[1] + "]"); // 输出: [0,1]

        // 示例 2
        int[] nums2 = {3, 2, 4};
        int target2 = 6;
        int[] result2 = solution.twoSum(nums2, target2);
        System.out.println("示例 2 输出: [" + result2[0] + "," + result2[1] + "]"); // 输出: [1,2]

        // 示例 3
        int[] nums3 = {3, 3};
        int target3 = 6;
        int[] result3 = solution.twoSum(nums3, target3);
        System.out.println("示例 3 输出: [" + result3[0] + "," + result3[1] + "]"); // 输出: [0,1]
    }
}
```
