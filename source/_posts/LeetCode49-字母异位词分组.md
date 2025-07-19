---
title: LeetCode49-字母异位词分组
tags:
  - 算法
  - LeetCode
  - 哈希表
categories:
  - 算法
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250719143206741.png'
toc: true
abbrlink: 25f67f4e
date: 2025-07-19 14:22:29
---

## 题目

给你一个字符串数组，请你将 字母异位词 组合在一起。可以按任意顺序返回结果列表。

**示例 1:**
- 输入: strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
- 输出: [["bat"],["nat","tan"],["ate","eat","tea"]]
- 解释：
> 在 strs 中没有字符串可以通过重新排列来形成 "bat"。
> 
> 字符串 "nat" 和 "tan" 是字母异位词，因为它们可以重新排列以形成彼此。
> 
> 字符串 "ate" ，"eat" 和 "tea" 是字母异位词，因为它们可以重新排列以形成彼此。

**示例 2:**
- 输入: strs = [""]
- 输出: [[""]]

**示例 3:**
- 输入: strs = ["a"]
- 输出: [["a"]]

**提示：**
- 1 <= strs.length <= 104
- 0 <= strs[i].length <= 100
- strs[i] 仅包含小写字母

## 分析

### 1\. 解题思路

字母异位词（Anagrams）指的是由相同字母以不同顺序组成的单词。例如，"eat"、"tea" 和 "ate" 互为字母异位词，因为它们都包含一个 'a'、一个 'e' 和一个 't'。

要将它们分组，我们需要一个“标准”或者说“钥匙”（Key），让所有互为字母异位词的字符串都能映射到同一个钥匙上。而非字母异位词的字符串则映射到不同的钥匙上。

一个很自然的想法是：**如果两个字符串是字母异位词，那么将它们的字符排序后，得到的字符串一定是相同的。**

例如：

* "eat" -\> 排序后 -\> "aet"
* "tea" -\> 排序后 -\> "aet"
* "ate" -\> 排序后 -\> "aet"
* "tan" -\> 排序后 -\> "ant"
* "nat" -\> 排序后 -\> "ant"
* "bat" -\> 排序后 -\> "abt"

这样，"aet"、"ant"、"abt" 就成了我们需要的“钥匙”。

我们可以利用哈希表（`HashMap`）来解决这个问题：

1.  遍历输入的字符串数组 `strs`。
2.  对于每个字符串，将其字符进行排序，生成一个唯一的钥匙。
3.  使用这个钥匙在哈希表中查找对应的分组（一个 `List<String>`）。
4.  如果找到了分组，就将当前（未排序的）原始字符串加入该分组。
5.  如果没有找到，就为这个新钥匙创建一个新的分组，并将当前原始字符串加进去。
6.  遍历结束后，哈希表中所有的值（Value）就是我们需要的全部分组。

### 2\. 备选思路：计数

除了排序，另一种生成钥匙的方法是计数。因为题目说明了字符串仅包含小写字母，我们可以用一个长度为 26 的数组来统计每个字符串中 a-z 各个字母出现的次数。然后将这个计数数组转换成一个唯一的字符串作为钥匙。

例如，对于 "eat"：

* 计数数组为 `[1, 0, 0, 0, 1, 0, ..., 1, ...]` (a=1, e=1, t=1)
* 可以生成钥匙，如 `"1#0#0#0#1#...#1"`（用 `#` 分隔避免歧义，例如1和12会被区分为`1#`和`12#`）。

这种方法在理论上时间复杂度更优，因为计数的时间复杂度是 $O(K)$（K为字符串长度），而排序是 $O(K \\log K)$。在下面的代码实现中，我们将采用更直观、编码更简洁的排序法。

-----

### 代码逻辑详解

1.  **初始化**：我们创建一个 `HashMap<String, List<String>>`。这个 `Map` 的键（Key）将是排序后的字符串，值（Value）是一个列表，存放所有能排序成这个键的原始字符串。
2.  **遍历**：我们使用 for-each 循环遍历输入数组 `strs` 中的每一个字符串 `s`。
3.  **生成键**：
    * `s.toCharArray()` 将字符串 `s` 转换为一个字符数组。
    * `Arrays.sort(charArray)` 对这个字符数组进行原地排序。
    * `new String(charArray)` 将排序后的字符数组重新组合成一个新的字符串 `key`。这个 `key` 就是这组字母异位词的唯一标识。
4.  **分组**：
    * `map.computeIfAbsent(key, k -> new ArrayList<>())` 是一个非常高效的写法。它会去 `map` 中查找 `key`。
        * 如果 `key` **不存在**，它会执行 `k -> new ArrayList<>()` 这个 Lambda 表达式，创建一个新的空列表，并将 `(key, newList)` 这个键值对存入 `map`，最后返回这个新列表。
        * 如果 `key` **已存在**，它会直接返回 `key` 对应的那个已存在的列表。
    * 拿到列表后，我们调用 `.add(s)` 将当前的**原始字符串 `s`** 添加到这个列表中。
5.  **返回结果**：当循环结束后，`map` 中已经按字母异位词分好组了。`map.values()` 方法会返回一个包含 `map` 中所有值的集合（`Collection<List<String>>`）。我们将其转换为 `ArrayList` 并返回，这正是题目要求的 `List<List<String>>` 格式。

### 复杂度分析

假设输入字符串数组的长度为 $N$，数组中最长字符串的长度为 $K$。

* **时间复杂度: $O(N \\cdot K \\log K)$**

    * 我们需要遍历 $N$ 个字符串。
    * 对于每个字符串，我们需要花费时间来生成它的键。主要开销在于排序，对一个长度为 $K$ 的字符串进行排序的时间复杂度是 $O(K \\log K)$。
    * 哈希表的插入和查找操作平均时间复杂度接近 $O(1)$（这里键的长度是 $K$，所以严格来说是 $O(K)$，但 $K \\log K$ 是主导）。
    * 因此，总的时间复杂度是 $N$ 次排序操作，即 $O(N \\cdot K \\log K)$。

* **空间复杂度: $O(N \\cdot K)$**

    * 我们需要一个哈希表来存储所有的字符串。在最坏的情况下（所有字符串都不是字母异位词），哈希表需要存储 $N$ 个键和 $N$ 个原始字符串。
    * 所有字符串的总字符数最多是 $N \\times K$。因此，存储这些字符串所需的空间是 $O(N \\cdot K)$。这是空间复杂度的主要部分。

## 答案

```java
import java.util.*;

/**
 * LeetCode 49. 字母异位词分组
 */
class Solution {
    /**
     * 将字符串数组中的字母异位词组合在一起。
     *
     * @param strs 输入的字符串数组
     * @return 按字母异位词分组后的列表
     */
    public List<List<String>> groupAnagrams(String[] strs) {
        // 处理边界情况，如果输入为 null 或空数组，直接返回一个空列表
        if (strs == null || strs.length == 0) {
            return new ArrayList<>();
        }

        // 创建一个 HashMap，键是排序后的字符串（作为唯一标识），值是原始字符串的列表
        Map<String, List<String>> map = new HashMap<>();

        // 遍历输入的每一个字符串
        for (String s : strs) {
            // 将当前字符串转换为字符数组，以便进行排序
            char[] charArray = s.toCharArray();
            // 对字符数组进行排序
            Arrays.sort(charArray);
            // 将排序后的字符数组转换回字符串，作为哈希表的键
            String key = new String(charArray);

            // 检查哈希表中是否已存在该键
            // 如果不存在，computeIfAbsent 方法会创建一个新的 ArrayList 并放入 map 中
            // 然后返回这个新创建的或已存在的列表
            List<String> anagramGroup = map.computeIfAbsent(key, k -> new ArrayList<>());
            
            // 将原始字符串 s 添加到对应的分组中
            anagramGroup.add(s);
        }

        // 哈希表的所有值（即所有的分组列表）就是最终结果
        // 将 map.values() 转换为一个新的 ArrayList 并返回
        return new ArrayList<>(map.values());
    }
}

// 主类，用于测试
public class Main {
    public static void main(String[] args) {
        Solution solution = new Solution();

        // 示例 1
        String[] strs1 = {"eat", "tea", "tan", "ate", "nat", "bat"};
        List<List<String>> result1 = solution.groupAnagrams(strs1);
        System.out.println("示例 1 输出: " + result1);
        // 预期输出: [[bat],[nat,tan],[ate,eat,tea]] (顺序可能不同)

        // 示例 2
        String[] strs2 = {""};
        List<List<String>> result2 = solution.groupAnagrams(strs2);
        System.out.println("示例 2 输出: " + result2);
        // 预期输出: [[]]

        // 示例 3
        String[] strs3 = {"a"};
        List<List<String>> result3 = solution.groupAnagrams(strs3);
        System.out.println("示例 3 输出: " + result3);
        // 预期输出: [[a]]
    }
}
```
