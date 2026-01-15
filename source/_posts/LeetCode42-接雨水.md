---
title: LeetCode42-接雨水
abbrlink: 625d00ec
date: '2025-07-20T18:21:27.000Z'
tags:
  - 算法
categories:
  - 算法
toc: true
---


## 题目

给定 `n` 个非负整数表示每个宽度为 `1` 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。

**示例 1：**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250720181923443.png)

- 输入：`height = [0,1,0,2,1,0,1,3,2,1,2,1]`
- 输出：`6`
- 解释：上面是由数组 `[0,1,0,2,1,0,1,3,2,1,2,1]` 表示的高度图，在这种情况下，可以接 `6` 个单位的雨水（蓝色部分表示雨水）。

**示例 2：**

- 输入：`height = [4,2,0,3,2,5]`
- 输出：9

**提示：**

- `n == height.length`
- `1 <= n <= 2 * 104`
- `0 <= height[i] <= 105`

## 分析

题目要求我们计算一个由非负整数数组表示的高度图能接多少雨水。我们可以把数组的每个元素想象成一个宽度为 1 的柱子。

要计算能接多少雨水，关键在于找出每个位置上方能够蓄水的高度。对于数组中的任意一个位置 `i`，其上方能蓄水的高度取决于它**左边的最高柱子**和**右边的最高柱子**中**较矮**的那个。

我们用 `max_left[i]` 表示位置 `i` 左边（包括 `i`）的最高柱子高度，用 `max_right[i]` 表示位置 `i` 右边（包括 `i`）的最高柱子高度。那么，在位置 `i` 能蓄的水量就是：

`water[i] = min(max_left[i], max_right[i]) - height[i]`

这个公式的含义是，位置 `i` 的水面高度由其左右两边的“堤坝”中较矮的那个决定，再减去当前位置柱子本身的高度，就是该位置的积水深度。

将所有位置的积水量相加，就是总的雨水量。

`total_water = sum(water[i])` for all `i`.

虽然我们可以通过两次遍历（一次从左到右计算 `max_left`，一次从右到左计算 `max_right`），然后再遍历一次来计算总和，但这需要 $O(N)$ 的额外空间。题目要求使用**双指针**的思路，这可以让我们在 $O(1)$ 的额外空间内解决问题。

双指针方法的核心思想是：通过维护左右两个指针 `left` 和 `right`，以及从左到右和从右到左的最高柱子高度 `left_max` 和 `right_max`，来逐步计算总雨水量。

1.  **初始化指针和变量**:

    * 创建一个左指针 `left`，指向数组的开头 (`index = 0`)。
    * 创建一个右指针 `right`，指向数组的末尾 (`index = n-1`)。
    * 创建 `left_max` 用于记录 `height[0...left]` 中的最大值，初始为 0。
    * 创建 `right_max` 用于记录 `height[right...n-1]` 中的最大值，初始为 0。
    * 创建 `ans` 用于累加总雨水量，初始为 0。

2.  **移动指针并计算**:

    * 当 `left < right` 时，循环继续。
    * 在循环的每一步，我们比较 `height[left]` 和 `height[right]` 的高度。
    * **情况一：`height[left] < height[right]`**
        * 此时，我们处理左指针 `left`。
        * 因为右边有一个更高的柱子 `height[right]` 作为“堤坝”，所以位置 `left` 能接的雨水高度就取决于它左边的最高柱子 `left_max`。
        * 我们更新 `left_max`：`left_max = max(left_max, height[left])`。
        * 然后计算位置 `left` 的积水：`ans += left_max - height[left]`。
        * 将左指针向右移动一位：`left++`。
    * **情况二：`height[left] >= height[right]`**
        * 此时，我们处理右指针 `right`。
        * 因为左边有一个不低于当前右边的柱子 `height[left]` 作为“堤坝”，所以位置 `right` 能接的雨水高度就取决于它右边的最高柱子 `right_max`。
        * 我们更新 `right_max`：`right_max = max(right_max, height[right])`。
        * 然后计算位置 `right` 的积水：`ans += right_max - height[right]`。
        * 将右指针向左移动一位：`right--`。

3.  **循环结束**:

    * 当 `left` 和 `right` 相遇时，循环结束。此时所有可能积水的位置都已被计算完毕。
    * 返回 `ans`。

**为什么这个逻辑是正确的？**
关键在于，当我们处理 `left` 指针时（因为 `height[left] < height[right]`），我们知道 `left_max` 是 `left` 左边的最高点，但我们不确定 `right_max` 是不是 `left` 右边的最高点。然而，我们能确定的是 `height[right]` 以及它右边的 `right_max` 至少都比 `height[left]` 高。所以，对于位置 `left` 而言，决定其蓄水量的瓶颈（短板）一定在左边，即 `left_max`。因此，可以直接根据 `left_max` 来计算积水。反之亦然。

这种方法巧妙地避免了对每个点都去寻找完整的左右最高点，而是在移动指针的过程中，动态地确定了每个位置的“有效”堤坝高度，从而一次遍历就解决了问题。

## 答案

```java
class Solution {
    /**
     * 使用双指针计算接雨水问题
     *
     * @param height 代表柱子高度的非负整数数组
     * @return 可以接的雨水总量
     */
    public int trap(int[] height) {
        // 处理边界情况，如果数组长度小于3，不可能接到雨水
        if (height == null || height.length < 3) {
            return 0;
        }

        int n = height.length;
        int left = 0, right = n - 1; // 初始化左右指针
        int leftMax = 0, rightMax = 0; // 初始化左右两边的最大高度
        int ans = 0; // 最终结果

        // 当左右指针相遇时，遍历结束
        while (left < right) {
            // 从左边开始处理
            if (height[left] < height[right]) {
                // 如果当前左边柱子的高度比 leftMax 小，说明可以蓄水
                if (height[left] >= leftMax) {
                    // 否则，它成为了新的左边最高点
                    leftMax = height[left];
                } else {
                    // 计算当前位置的蓄水量并累加
                    ans += (leftMax - height[left]);
                }
                // 左指针向右移动
                left++;
            }
            // 从右边开始处理
            else { // height[left] >= height[right]
                // 如果当前右边柱子的高度比 rightMax 小，说明可以蓄水
                if (height[right] >= rightMax) {
                    // 否则，它成为了新的右边最高点
                    rightMax = height[right];
                } else {
                    // 计算当前位置的蓄水量并累加
                    ans += (rightMax - height[right]);
                }
                // 右指针向左移动
                right--;
            }
        }

        return ans;
    }
}

// 主函数用于测试
public class Main {
    public static void main(String[] args) {
        Solution solution = new Solution();

        // 示例 1
        int[] height1 = {0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1};
        System.out.println("示例 1 输入: [0,1,0,2,1,0,1,3,2,1,2,1]");
        System.out.println("示例 1 输出: " + solution.trap(height1)); // 应该输出 6

        // 示例 2
        int[] height2 = {4, 2, 0, 3, 2, 5};
        System.out.println("示例 2 输入: [4,2,0,3,2,5]");
        System.out.println("示例 2 输出: " + solution.trap(height2)); // 应该输出 9
    }
}
```