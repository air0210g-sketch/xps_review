# 算法面试通关指南 (基于 LeetCode Hot 100)

> **核心理念**：不背题，重思维。即使是“背”，也是背后的**模式 (Pattern)** 和 **第一性原理 (First Principles)**。
> **目标**：通过 Hot 100 题目的专项训练，掌握通用解题范式，具备举一反三的能力。

---

## 📚 学习路线与建议

1.  **分门别类，专项突破**：不要随机刷题。按“链表 -> 二叉树 -> 数组/哈希 -> 双指针 -> 动态规划”的顺序进行。
2.  **注重模板**：对于基础算法（如二分、BFS、反转链表），必须烂熟于心，达到 muscle memory（肌肉记忆）的程度。
3.  **Think out loud**：练习时尝试用语言描述思路，模拟面试场景。

---

## 🧠 核心思维：递归与分治 (Recursion & Divide and Conquer)

> **"相信函数契约" (Trust the Function Contract)**：不要人脑模拟递归栈，而是假设子问题已经被函数解决，专注于“当前层”的逻辑。

### 🧩 通用解题模板
func recursion(problem any) Result {
    // 1. Base Case (终止条件)
    if isSmall(problem) {
        return result
    }

    // 2. Divide (分解)
    subProblems := split(problem)

    // 3. Conquer (递归求解)
    // 假设 recursion 函数能正确解决子问题
    res1 := recursion(subProblems[0])
    res2 := recursion(subProblems[1])

    // 4. Merge (合并)
    return merge(res1, res2)
}

### 💡 实例：链表排序 (Merge Sort)
以 **[148. 排序链表](https://leetcode.cn/problems/sort-list/)** 为例，完美契合分治模板：

1.  **Base Case**: 链表为空或只有一个节点 -> 直接返回。
2.  **Divide**: 快慢指针找中点，切断链表 -> 得到 `left`, `right` 两段。
3.  **Conquer**: `sortList(left)` 和 `sortList(right)` -> 得到两个**有序**链表。
4.  **Merge**: 合并两个有序链表 (`mergeTwoLists`) -> 得到最终结果。

---

## 🔗 专题一：链表 (Linked List)

### 💡 核心思想
链表问题难点在于指针操作容易出错（丢节点、环、NPE）。
*   **虚拟头节点 (Dummy Node)**：无脑上 Dummy Node 解决头节点变动问题（如删除、合并）。
*   **快慢指针**：解决环检测、找中点、倒数第 K 个节点。

### 🧩 通用模板 (反转链表)
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head
    for curr != nil {
        nxt := curr.Next
        curr.Next = prev
        prev = curr
        curr = nxt
    }
    return prev
}

### 🔥 Hot 100 全量列表
*   **[160. 相交链表](https://leetcode.cn/problems/intersection-of-two-linked-lists/)** (简单) - A+B=B+A 双指针相遇。
*   **[206. 反转链表](https://leetcode.cn/problems/reverse-linked-list/)** (简单) - 基础题。
*   **[234. 回文链表](https://leetcode.cn/problems/palindrome-linked-list/)** (简单) - 快慢指针找中点 -> 反转后半部分 -> 比较。
*   **[21. 合并两个有序链表](https://leetcode.cn/problems/merge-two-sorted-lists/)** (简单) - 递归或迭代 + Dummy Node。
*   **[141. 环形链表](https://leetcode.cn/problems/linked-list-cycle/)** (简单) - 快慢指针。
*   **[142. 环形链表 II](https://leetcode.cn/problems/linked-list-cycle-ii/)** (中等) - 快慢指针相遇后，慢指针回拨。
*   **[19. 删除链表的倒数第 N 个结点](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)** (中等) - 快指针先走 N 步。
*   **[24. 两两交换链表中的节点](https://leetcode.cn/problems/swap-nodes-in-pairs/)** (中等) - 递归或迭代。
*   **[2. 两数相加](https://leetcode.cn/problems/add-two-numbers/)** (中等) - 模拟加法，注意进位。
*   **[148. 排序链表](https://leetcode.cn/problems/sort-list/)** (中等) - 归并排序 (找中点->断开->递归->合并)。
*   **[23. 合并 K 个升序链表](https://leetcode.cn/problems/merge-k-sorted-lists/)** (困难) - 优先队列 (小顶堆) 或 分治合并。
*   **[25. K 个一组翻转链表](https://leetcode.cn/problems/reverse-nodes-in-k-group/)** (困难) - 模拟，需大量指针操作。
*   **[146. LRU 缓存](https://leetcode.cn/problems/lru-cache/)** (中等) - 哈希表 + 双向链表 (重点)。

---

## 🌲 专题二：二叉树 (Binary Tree)

### 💡 核心思想
*   **DFS (递归)**：前序 (根左右)、中序 (左根右)、后序 (左右根)。思考：需要子树返回什么？
*   **BFS (层序)**：利用队列 (Queue)。

### 🧩 通用模板 (BFS)
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return [][]int{}
    }
    
    var res [][]int
    queue := []*TreeNode{root}
    
    for len(queue) > 0 {
        var level []int
        size := len(queue) // 固定当前层的大小
        
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:] // 出队
            
            level = append(level, node.Val)
            
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        res = append(res, level)
    }
    return res
}

### 🔥 Hot 100 全量列表
*   **[94. 二叉树的中序遍历](https://leetcode.cn/problems/binary-tree-inorder-traversal/)** (简单)
*   **[104. 二叉树的最大深度](https://leetcode.cn/problems/maximum-depth-of-binary-tree/)** (简单)
*   **[226. 翻转二叉树](https://leetcode.cn/problems/invert-binary-tree/)** (简单)
*   **[101. 对称二叉树](https://leetcode.cn/problems/symmetric-tree/)** (简单)
*   **[543. 二叉树的直径](https://leetcode.cn/problems/diameter-of-binary-tree/)** (简单) - 后序遍历，边长 = 左高+右高。
*   **[617. 合并二叉树](https://leetcode.cn/problems/merge-two-binary-trees/)** (简单)
*   **[108. 将有序数组转换为二叉搜索树](https://leetcode.cn/problems/convert-sorted-array-to-binary-search-tree/)** (简单) - 找中点作为根，递归构建。
*   **[102. 二叉树的层序遍历](https://leetcode.cn/problems/binary-tree-level-order-traversal/)** (中等)
*   **[98. 验证二叉搜索树](https://leetcode.cn/problems/validate-binary-search-tree/)** (中等) - 中序遍历是升序，或者递归带范围 (min, max)。
*   **[199. 二叉树的右视图](https://leetcode.cn/problems/binary-tree-right-side-view/)** (中等) - BFS 取每层最后一个，或 DFS (根右左)。
*   **[114. 二叉树展开为链表](https://leetcode.cn/problems/flatten-binary-tree-to-linked-list/)** (中等) - 后序遍历变形。
*   **[105. 从前序与中序遍历序列构造二叉树](https://leetcode.cn/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)** (中等)
*   **[437. 路径总和 III](https://leetcode.cn/problems/path-sum-iii/)** (中等) - 前缀和 + DFS。
*   **[236. 二叉树的最近公共祖先](https://leetcode.cn/problems/lowest-common-ancestor-of-a-binary-tree/)** (中等) - 经典递归：左空返右，右空返左，都不空返根。
*   **[124. 二叉树中的最大路径和](https://leetcode.cn/problems/binary-tree-maximum-path-sum/)** (困难) - 后序遍历，维护全局 Max。

---

## 🔢 专题三：数组与哈希 (Array & Hash)

### 💡 核心思想
*   **哈希表**：O(1) 查询。
*   **前缀和**：子数组和问题。
*   **矩阵技巧**：螺旋打印、置零。

### 🔥 Hot 100 全量列表
*   **[1. 两数之和](https://leetcode.cn/problems/two-sum/)** (简单)
*   **[49. 字母异位词分组](https://leetcode.cn/problems/group-anagrams/)** (中等)
*   **[128. 最长连续序列](https://leetcode.cn/problems/longest-consecutive-sequence/)** (中等) - Set 去重，找起点。
*   **[238. 除自身以外数组的乘积](https://leetcode.cn/problems/product-of-array-except-self/)** (中等) - 前缀积 * 后缀积。
*   **[560. 和为 K 的子数组](https://leetcode.cn/problems/subarray-sum-equals-k/)** (中等) - 前缀和 + 哈希表。
*   **[41. 缺失的第一个正数](https://leetcode.cn/problems/first-missing-positive/)** (困难) - 原地哈希，把 nums[i] 放到 nums[i]-1 的位置。
*   **[73. 矩阵置零](https://leetcode.cn/problems/set-matrix-zeroes/)** (中等) - 用第一行/第一列标记。
*   **[54. 螺旋矩阵](https://leetcode.cn/problems/spiral-matrix/)** (中等) - 模拟边界收缩。
*   **[48. 旋转图像](https://leetcode.cn/problems/rotate-image/)** (中等) - 先对角翻转，再左右翻转。
*   **[240. 搜索二维矩阵 II](https://leetcode.cn/problems/search-a-2d-matrix-ii/)** (中等) - 从右上角或左下角开始搜。

---

## 🪟 专题四：双指针与滑动窗口 (Two Pointers)

### 💡 核心思想
*   **滑动窗口**：求最长/最短子串。
*   **双指针**：接雨水、三数之和。

### 🔥 Hot 100 全量列表
*   **[283. 移动零](https://leetcode.cn/problems/move-zeroes/)** (简单)
*   **[11. 盛最多水的容器](https://leetcode.cn/problems/container-with-most-water/)** (中等)
*   **[15. 三数之和](https://leetcode.cn/problems/3sum/)** (中等)
*   **[3. 无重复字符的最长子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/)** (中等)
*   **[438. 找到字符串中所有字母异位词](https://leetcode.cn/problems/find-all-anagrams-in-a-string/)** (中等) - 固定窗口大小的滑动窗口。
*   **[76. 最小覆盖子串](https://leetcode.cn/problems/minimum-window-substring/)** (困难) - 滑动窗口 (need/have 计数)。
*   **[42. 接雨水](https://leetcode.cn/problems/trapping-rain-water/)** (困难) - 双指针 (按列求) 或 单调栈 (按行求)。

---

## 📈 专题五：动态规划 (Dynamic Programming)

### 💡 核心思想
*   **一维 DP**：爬楼梯、打家劫舍、LIS。
*   **二维 DP**：最长公共子序列、编辑距离、背包。

### 🔥 Hot 100 全量列表
*   **[70. 爬楼梯](https://leetcode.cn/problems/climbing-stairs/)** (简单)
*   **[118. 杨辉三角](https://leetcode.cn/problems/pascals-triangle/)** (简单)
*   **[198. 打家劫舍](https://leetcode.cn/problems/house-robber/)** (中等)
*   **[279. 完全平方数](https://leetcode.cn/problems/perfect-squares/)** (中等) - 类似零钱兑换。
*   **[322. 零钱兑换](https://leetcode.cn/problems/coin-change/)** (中等)
*   **[139. 单词拆分](https://leetcode.cn/problems/word-break/)** (中等) - `dp[i]` = `dp[j]` && `s[j:i]` in dict。
*   **[300. 最长递增子序列](https://leetcode.cn/problems/longest-increasing-subsequence/)** (中等)
*   **[152. 乘积最大子数组](https://leetcode.cn/problems/maximum-product-subarray/)** (中等) - 维护 max 和 min (负负得正)。
*   **[416. 分割等和子集](https://leetcode.cn/problems/partition-equal-subset-sum/)** (中等) - 0-1 背包问题。
*   **[32. 最长有效括号](https://leetcode.cn/problems/longest-valid-parentheses/)** (困难) - 栈 或 DP。
*   **[62. 不同路径](https://leetcode.cn/problems/unique-paths/)** (中等)
*   **[64. 最小路径和](https://leetcode.cn/problems/minimum-path-sum/)** (中等)
*   **[5. 最长回文子串](https://leetcode.cn/problems/longest-palindromic-substring/)** (中等)
*   **[1143. 最长公共子序列](https://leetcode.cn/problems/longest-common-subsequence/)** (中等) - `if s1[i]==s2[j] then dp[i-1][j-1]+1`。
*   **[72. 编辑距离](https://leetcode.cn/problems/edit-distance/)** (困难) - 增删改三选一。

---

## 🔙 专题六：回溯算法 (Backtracking)

### 💡 核心思想
做选择 -> 递归 -> 撤销选择。

### 🔥 Hot 100 全量列表
*   **[46. 全排列](https://leetcode.cn/problems/permutations/)** (中等)
*   **[78. 子集](https://leetcode.cn/problems/subsets/)** (中等)
*   **[17. 电话号码的字母组合](https://leetcode.cn/problems/letter-combinations-of-a-phone-number/)** (中等)
*   **[39. 组合总和](https://leetcode.cn/problems/combination-sum/)** (中等)
*   **[22. 括号生成](https://leetcode.cn/problems/generate-parentheses/)** (中等)
*   **[79. 单词搜索](https://leetcode.cn/problems/word-search/)** (中等) - 网格 DFS。
*   **[131. 分割回文串](https://leetcode.cn/problems/palindrome-partitioning/)** (中等) - 切割问题。
*   **[51. N 皇后](https://leetcode.cn/problems/n-queens/)** (困难) - 经典回溯。

---

## 📚 专题七：栈与堆 (Stack & Heap)

### 💡 核心思想
*   **单调栈**：找左右第一个大小值。
*   **Top K**：堆。

### 🔥 Hot 100 全量列表
*   **[20. 有效的括号](https://leetcode.cn/problems/valid-parentheses/)** (简单)
*   **[155. 最小栈](https://leetcode.cn/problems/min-stack/)** (中等)
*   **[394. 字符串解码](https://leetcode.cn/problems/decode-string/)** (中等)
*   **[739. 每日温度](https://leetcode.cn/problems/daily-temperatures/)** (中等) - 单调栈经典。
*   **[84. 柱状图中最大的矩形](https://leetcode.cn/problems/largest-rectangle-in-histogram/)** (困难) - 单调栈 (左右边界)。
*   **[215. 数组中的第K个最大元素](https://leetcode.cn/problems/kth-largest-element-in-an-array/)** (中等)
*   **[347. 前 K 个高频元素](https://leetcode.cn/problems/top-k-frequent-elements/)** (中等)
*   **[295. 数据流的中位数](https://leetcode.cn/problems/find-median-from-data-stream/)** (困难)

---

## 🗺️ 专题八：图论 (Graph)

### 💡 核心思想
*   **BFS**：最短路、层层扩散。
*   **DFS**：连通性、岛屿问题。
*   **拓扑排序**：依赖关系 (课程表)。

### 🔥 Hot 100 全量列表
*   **[200. 岛屿数量](https://leetcode.cn/problems/number-of-islands/)** (中等) - 沉岛思想 (DFS/BFS)。
*   **[994. 腐烂的橘子](https://leetcode.cn/problems/rotting-oranges/)** (中等) - 多源 BFS。
*   **[207. 课程表](https://leetcode.cn/problems/course-schedule/)** (中等) - 拓扑排序 (入度表 + Queue) 或 DFS 找环。
*   **[208. 实现 Trie (前缀树)](https://leetcode.cn/problems/implement-trie-prefix-tree/)** (中等) - 多叉树结构。

---

## 💰 专题九：贪心与技巧 (Greedy & Skills)

### 💡 核心思想
*   **贪心**：局部最优推出全局最优。
*   **二分查找**：有序数组，或“答案有单调性”。

### 🔥 Hot 100 全量列表
*   **[121. 买卖股票的最佳时机](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock/)** (简单) - 维护 minPrice。
*   **[55. 跳跃游戏](https://leetcode.cn/problems/jump-game/)** (中等) - 维护 maxReachable。
*   **[45. 跳跃游戏 II](https://leetcode.cn/problems/jump-game-ii/)** (中等) - 每次跳到最远能覆盖范围内。
*   **[763. 划分字母区间](https://leetcode.cn/problems/partition-labels/)** (中等) - 记录每个字符最后出现的位置。
*   **[136. 只出现一次的数字](https://leetcode.cn/problems/single-number/)** (简单) - 异或 (XOR) 操作。
*   **[169. 多数元素](https://leetcode.cn/problems/majority-element/)** (简单) - 摩尔投票法 (抵消)。
*   **[75. 颜色分类](https://leetcode.cn/problems/sort-colors/)** (中等) - 荷兰国旗问题 (三指针)。
*   **[31. 下一个排列](https://leetcode.cn/problems/next-permutation/)** (中等) - 从后向前找第一个降序，交换，反转。
*   **[287. 寻找重复数](https://leetcode.cn/problems/find-the-duplicate-number/)** (中等) - 快慢指针 (数组视作链表，值即 Next 指针)。
*   **[35. 搜索插入位置](https://leetcode.cn/problems/search-insert-position/)** (简单) - 二分查找。
*   **[34. 在排序数组中查找元素的第一个和最后一个位置](https://leetcode.cn/problems/find-first-and-last-position-of-element-in-sorted-array/)** (中等) - 两次二分。
*   **[33. 搜索旋转排序数组](https://leetcode.cn/problems/search-in-rotated-sorted-array/)** (中等) - 根据 mid 与 left/right 关系判断哪半边有序。
*   **[153. 寻找旋转排序数组中的最小值](https://leetcode.cn/problems/find-minimum-in-rotated-sorted-array/)** (中等)
*   **[4. 寻找两个正序数组的中位数](https://leetcode.cn/problems/median-of-two-sorted-arrays/)** (困难) - 二分查找第 K 小元素。

---

## 📝 总结
以上即为 LeetCode Hot 100 的全量题目分类整理。
**建议执行方案**：
1.  **第一轮**：每个专题挑 2-3 道简单/中等题寻找感觉。
2.  **第二轮**：按专题刷完所有中等题。
3.  **第三轮**：攻克困难题，或二刷错题。
