---
title: LeetCode128-最长连续序列
tags:
  - 算法
  - LeetCode
  - 哈希
categories:
  - 算法
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250719142259466.png'
toc: true
abbrlink: 3f27087f
date: 2025-07-19 14:23:47
---

## 题目

给定一个未排序的整数数组 nums ，找出数字连续的最长序列（不要求序列元素在原数组中连续）的长度。
请你设计并实现时间复杂度为 O(n) 的算法解决此问题。

**示例 1：**
- 输入：nums = [100,4,200,1,3,2]
- 输出：4
- 解释：最长数字连续序列是 [1, 2, 3, 4]。它的长度为 4。

**示例 2：**
- 输入：nums = [0,3,7,2,5,8,4,6,0,1]
- 输出：9

**示例 3：**

- 输入：nums = [1,0,1,2]
- 输出：3


**提示：**

- 0 <= nums.length <= 105
- -109 <= nums[i] <= 109

## 分析


### 1. 初步思考与暴力解法

最直观的想法是，对于数组中的每一个数 `x`，我们都去尝试寻找 `x+1`, `x+2`, `x+3`, ... 是否存在于数组中，直到找不到为止，以此来确定以 `x` 开头的连续序列的长度。然后我们遍历所有数字，找出最长的那个长度。

这个方法需要两层循环：外层循环遍历每个数字 `x`，内层循环则去数组中查找 `x+1`, `x+2` 等。在未排序的数组中查找一个元素的时间复杂度是 O(n)，因此总的时间复杂度会是 O(n^2) 甚至 O(n^3)，这不满足题目 O(n) 的要求。

### 2. 排序法

一个改进的思路是先对数组进行排序。

* **思路**:

    1.  对原数组 `nums`进行排序。
    2.  遍历排序后的数组，用一个变量 `currentStreak` 记录当前连续序列的长度，用另一个变量 `longestStreak` 记录全局最长的连续序列长度。
    3.  如果当前数字比前一个数字大 1，则 `currentStreak` 加 1。
    4.  如果当前数字与前一个数字相等，说明是重复数字，跳过即可，不影响连续性。
    5.  如果当前数字比前一个数字大超过 1，说明连续序列中断了，重置 `currentStreak`为 1。
    6.  每次更新 `currentStreak` 后，都用它来更新 `longestStreak`。

* **复杂度分析**:

    * 时间复杂度：`O(n \log n)`。主要瓶颈在于排序。
    * 空间复杂度：`O(1)` 或 `O(\log n)`，取决于排序算法内部使用的空间。

* **结论**: 这个方法虽然比暴力法好，但时间复杂度仍然不满足题目要求的 O(n)。

### 3. 最优解法：哈希集合（HashSet）

为了达到 O(n) 的时间复杂度，我们需要一种能在 O(1) 时间内判断一个数是否存在的方法。这自然让我们想到了哈希表（在 Java 中是 `HashSet`）。

* **核心思想**:
  我们希望对一个连续序列，只在其起点（序列中最小的数）进行一次完整的遍历和计数。这样可以避免对序列中的其他数字（如 2, 3, 4）进行重复的计数工作。

* **如何判断一个数是序列的起点？**
  一个数 `num` 如果是连续序列的起点，那么 `num - 1` 这个数一定不存在于数组中。

* **算法步骤**:

    1.  **预处理**: 创建一个 `HashSet`，将数组 `nums` 中的所有数字都存入 `HashSet`。这一步有两个目的：
        * 去重：`HashSet` 自动处理了重复的数字。
        * 快速查找：提供了 O(1) 的平均时间复杂度的 `contains` 方法。
    2.  **遍历和计数**: 遍历 `HashSet` 中的每一个数字 `num`（或者遍历原数组 `nums` 也可以）。
    3.  **寻找起点**: 对于当前的数字 `num`，检查 `set.contains(num - 1)` 是否为 `false`。
        * 如果为 `true`，意味着 `num` 不是一个序列的起点（例如，对于序列 `[1, 2, 3, 4]`，当我们遍历到 `2` 时，因为 `1` 存在，所以 `2` 不是起点），我们直接跳过，不做任何处理。
        * 如果为 `false`，意味着 `num` 是一个连续序列的起点。我们就从 `num` 开始，不断检查 `num + 1`, `num + 2`, ... 是否存在于 `HashSet` 中，并计数，直到序列中断。
    4.  **更新最长长度**: 在找到一个序列的完整长度后，与全局的最长长度 `longestStreak` 进行比较并更新。
    5.  **返回结果**: 遍历完所有数字后，`longestStreak` 就是最终结果。

* **复杂度分析**:

    * **时间复杂度**: O(n)。
        * 将 `n` 个数字添加到 `HashSet` 中需要 O(n) 的时间。
        * 外层循环会执行 `n` 次（最多，如果无重复数字）。
        * 内层的 `while` 循环看起来像是嵌套循环，但需要注意的是，`while` 循环只会在一个数是序列起点时才会被执行。对于整个算法来说，每个数字最多只会被 `while` 循环访问一次（作为 `currentNum + 1`）。因此，所有 `while` 循环的总执行次数加起来也是 O(n) 级别的。
        * 总的时间复杂度是 O(n) + O(n) = O(n)。
    * **空间复杂度**: O(n)。我们需要一个 `HashSet` 来存储数组中所有的不重复元素。

* **结论**: 此方法在时间和空间上都满足题目的要求，是本题的最优解。

## 答案

```java
import java.util.HashSet;
import java.util.Set;

class Solution {
    /**
     * 找出数字连续的最长序列的长度。
     *
     * @param nums 未排序的整数数组
     * @return 最长连续序列的长度
     */
    public int longestConsecutive(int[] nums) {
        // 处理边界情况：如果数组为null或为空，直接返回0
        if (nums == null || nums.length == 0) {
            return 0;
        }

        // 1. 预处理：将所有数字存入HashSet，去重并提供O(1)的查找效率
        Set<Integer> numSet = new HashSet<>();
        for (int num : nums) {
            numSet.add(num);
        }

        int longestStreak = 0;

        // 2. 遍历集合中的每个数字
        for (int num : numSet) {
            // 3. 寻找起点：如果num-1不存在，则num可能是一个连续序列的起点
            //    这是本算法的核心优化，确保每个序列只从起点计算一次
            if (!numSet.contains(num - 1)) {
                int currentNum = num;
                int currentStreak = 1;

                // 4. 从起点开始，向后查找连续的数字
                while (numSet.contains(currentNum + 1)) {
                    currentNum += 1;
                    currentStreak += 1;
                }

                // 5. 更新全局最长序列的长度
                longestStreak = Math.max(longestStreak, currentStreak);
            }
        }

        return longestStreak;
    }
}

// 主函数用于测试
class Main {
    public static void main(String[] args) {
        Solution sol = new Solution();

        // 示例 1
        int[] nums1 = {100, 4, 200, 1, 3, 2};
        System.out.println("输入: [100,4,200,1,3,2]");
        System.out.println("输出: " + sol.longestConsecutive(nums1)); // 预期输出: 4

        // 示例 2
        int[] nums2 = {0, 3, 7, 2, 5, 8, 4, 6, 0, 1};
        System.out.println("输入: [0,3,7,2,5,8,4,6,0,1]");
        System.out.println("输出: " + sol.longestConsecutive(nums2)); // 预期输出: 9

        // 示例 3
        int[] nums3 = {1, 0, 1, 2};
        System.out.println("输入: [1,0,1,2]");
        System.out.println("输出: " + sol.longestConsecutive(nums3)); // 预期输出: 3

        // 边界情况：空数组
        int[] nums4 = {};
        System.out.println("输入: []");
        System.out.println("输出: " + sol.longestConsecutive(nums4)); // 预期输出: 0
    }
}
```
