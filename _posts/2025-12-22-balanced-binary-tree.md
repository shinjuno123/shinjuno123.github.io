---
title: "110. Balanced Binary Tree"
date: 2025-12-22 00:00:00 +0800
categories: [algorithm]
tags: [algorithm, leetcode]
---

# Problem

![LeetCodeProblem](/assets/2025-12-22-balanced-binary-tree.md/problem.png)


## How to solve problem
- Traverse through left and right, using DFS (Depth First Search)
- When there is no node found, return [True, 0]. True means it is balanced and 0 means its height. When the node is null, the height difference between nothings is 0. So it is balanced
- Check if the left and right are having balanced height. When left and right are presenet and left and right hight difference is less than or equal to 1, the current node has balanced tree.
- Return [balanced, 1 + the maximum height from the leaf node up to this point].

## Code
```py
# Definition for a binary tree node.
# class TreeNode:
#     def __init__(self, val=0, left=None, right=None):
#         self.val = val
#         self.left = left
#         self.right = right
class Solution:
    def isBalanced(self, root: Optional[TreeNode]) -> bool:
        
        def dfs(node):
            if not node:
                return [True, 0]
            
            left, right = dfs(node.left), dfs(node.right)
            balanced = left[0] and right[0] and abs(left[1] - right[1]) <= 1

            return [balanced, 1 + max(left[1], right[1])]


        return dfs(root)[0]
```

## Complexity
- Time Complexity: O(N) -> Traverse through all nodes
- Space Complexity: O(N) -> The worst case depth