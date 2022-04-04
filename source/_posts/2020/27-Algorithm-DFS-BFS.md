---
title: 算法中的 DFS 和 BFS
date: 2020-05-23 11:20:31
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/algorithm_training.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在总结数据结构与算法中关于深度优先遍历和广度优先遍历知识内容，学习如何套用模板轻松解决相关问题。

categories: 
  - 数据结构与算法

tags: 
  - 数据结构
  - 算法
---

## 背景描述

最近学习了 [极客大学--算法训练营](https://u.geekbang.org/subject/algorithm/1000343?utm_source=time_web&utm_medium=menu&utm_term=timewebmenu) 的课程，这一篇文章总结一下算法中的 `DFS` 和 `BFS` 相关知识内容。其实有关 `DFS` 和 `BFS` 的知识内容并不多，也不是非常难，类似于套公式一般直接套上去，就可以解决问题，不过在 [Leetcode](https://leetcode-cn.com/) 网站上刷题就会发现，的确很多都能够通过 `DFS` 或 `BFS` 来解决，但是有些题比较隐晦，需要进行思路转换才能抓住问题的关键。

## 深度优先遍历（DFS）

**深度优先搜索算法（英语：Depth-First-Search，DFS）** 是一种用于遍历或搜索树或图的算法。这个算法会尽可能深的搜索树的分支。当节点v的所在边都己被探寻过，搜索将回溯到发现节点v的那条边的起始节点。这一过程一直进行到已发现从源节点可达的所有节点为止。

`DFS` 是一种利用栈（Stack）或者递归实现的搜索算法。

### 递归实现

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Stack;

public class DepthFirstSearch {
    
    public List<Integer> dfs(TreeNode root) {
        List<Integer> result = new ArrayList<>();
        helper(root, result);
        return result;
    }
    
    private void helper(TreeNode node, List<Integer> result) {
        if (node == null) {
            return;
        }
        result.add(node.val);
        helper(node.left, result);
        helper(node.right, result);
    }
    
    public class TreeNode {
        public int val;
        public TreeNode left, right;
        public TreeNode(int val) {
            this.val= val;
        }
    }
}
```

### Stack 实现

```java
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;
import java.util.Queue;
import java.util.Stack;

public class DepthFirstSearch {
    
    public List<Integer> dfs(TreeNode root) {
        List<Integer> result = new ArrayList<>();
        Stack<TreeNode> stack = new Stack<>();
        stack.add(root);
        while (!stack.isEmpty()) {
            // 弹出栈顶元素
            TreeNode currentNode = stack.pop();
            result.add(currentNode.val);
            if (currentNode.right != null) {
                // 深度优先遍历，先遍历左边，后遍历右边，栈先进后出
                stack.push(currentNode.right);
            }
            if (currentNode.left != null) {
                stack.push(currentNode.left);
            }
        }
        return result;
    }
    
    public class TreeNode {
        public int val;
        public TreeNode left, right;
        public TreeNode(int val) {
            this.val= val;
        }
    }
}
```



## 广度优先遍历（BFS）

**广度优先搜索算法（英语：Breadth-First Search，缩写为BFS）**，又译作宽度优先搜索，或横向优先搜索，是一种图形搜索算法。简单的说，BFS是从根节点开始，沿着树的宽度遍历树的节点。如果所有节点均被访问，则算法中止。

`BFS` 是一种是一种利用队列（Queue）实现的搜索算法，其实现思想为：

- 队列不为空则循环；
- 将未访问的邻接点压入队列后面，然后从前面取出并访问（这样就做到了广度优先）。

### Queue 实现

```java
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;
import java.util.Queue;

public class BreadthFirstSearch {
    
    public List<Integer> bfs(TreeNode root) {
        List<Integer> result = new ArrayList<>();
        // 利用队列实现优先搜索算法
        Queue<TreeNode> queue = new LinkedList<>();
        queue.add(root);
        while (!queue.isEmpty()) {
            TreeNode currentNode = queue.poll();
            result.add(currentNode.val);
            if (currentNode.left != null) {
                queue.add(currentNode.left);
            }
            if (currentNode.right != null) {
                queue.add(currentNode.right);
            }
        }
        return result;
    }
    
    public class TreeNode {
        public int val;
        public TreeNode left, right;
        public TreeNode(int val) {
            this.val= val;
        }
    }
}
```

## 括号生成问题

### [22\. 括号生成](https://leetcode-cn.com/problems/generate-parentheses/)

Difficulty: **中等**


数字 _n_ 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且 **有效的** 括号组合。

**示例：**

```
输入：n = 3
输出：[
       "((()))",
       "(()())",
       "(())()",
       "()(())",
       "()()()"
     ]
```

这个题目乍一看和 `BFS、DFS` 完全是不沾边的，但是通过将具体题目过程进行抽象转换，也是可以用 `BFS、DFS` 实现的；具体思路是将要生成结果看做是 `root`，将左右括号使用的数量作为左右节点向外扩展，扩展到最后，左右括号数量都是 `0`，那么就得到了一个最终的结果。在扩展的过程中，如果一层层处理，那就是 `BFS` 了；如果是在一侧一直处理，处理完一侧再回溯继续处理，就是 `DFS` 了。

### 括号生成问题 DFS 代码（递归实现）

```java
class Solution {
    public List<String> generateParenthesis(int n) {
        List<String> result = new ArrayList<>();
        generateOneByOne("", result, n, n);
        return result;
    }
    
    private void generateOneByOne(String s, List<String> result, int left, int right) {
        if (left == 0 && right == 0) {
            result.add(s);
            return;
        }
        if (left > 0) {
            generateOneByOne(s + "(", result, left - 1, right);
        }
        if (right > left) {
            generateOneByOne(s + ")", result, left, right - 1);
        }
    }
}
```

### 括号生成问题 DFS 代码（Stack 实现）

```java
class Solution {

    class Node {
        private String res;
        private int left;
        private int right;
        public Node (String res, int left, int right) {
            this.res = res;
            this.left = left;
            this.right = right;
        }
    }
    
    public List<String> generateParenthesis(int n) {
        List<String> result = new ArrayList<>();
        if (n == 0) {
            return result;
        }
        Stack<Node> stack = new Stack<>();
        stack.add(new Node("", n, n));
        while (!stack.isEmpty()) {
            Node currentNode = stack.pop();
            if (currentNode.left == 0 && currentNode.right == 0) {
                result.add(currentNode.res);
            }
            if (currentNode.right > currentNode.left) {
                stack.push(new Node(currentNode.res + ")", currentNode.left, currentNode.right - 1));
            }
            if (currentNode.left > 0) {
                stack.push(new Node(currentNode.res + "(", currentNode.left - 1, currentNode.right));
            }
        }
        return result;
    }
}
```

### 括号生成问题 BFS 代码（Queue 实现）

```java
class Solution {

    class Node {
        private String res;
        private int left;
        private int right;
        public Node (String res, int left, int right) {
            this.res = res;
            this.left = left;
            this.right = right;
        }
    }

    public List<String> generateParenthesis(int n) {
        List<String> result = new ArrayList<>();
        if (n == 0) {
            return result;
        }
        Queue<Node> queue = new LinkedList<>();
        queue.add(new Node("", n, n));
        while (!queue.isEmpty()) {
            Node currentNode = queue.poll();
            if (currentNode.left == 0 && currentNode.right == 0) {
                result.add(currentNode.res);
            }
            if (currentNode.left > 0) {
                queue.add(new Node(currentNode.res + "(", currentNode.left - 1, currentNode.right));
            }
            if (currentNode.right > currentNode.left) {
                queue.add(new Node(currentNode.res + ")", currentNode.left, currentNode.right - 1));
            }
        }
        return result;
    }
}
```

## 其他题目（持续补充）

[102. 二叉树的层序遍历](https://leetcode-cn.com/problems/binary-tree-level-order-traversal/)

该题目本来是一道典型的 `BFS` 题目，但是在输出结果上有特殊的要求，所以要在循环队列时记录当前的层次，这样才能满足结果输出。

{% pdf https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/27-Algorithm-DFS-BFS/27-Algorithm-DFS-BFS-01.pdf %}

## 参考资料

- [广度优先搜索（需要梯子）](https://zh.wikipedia.org/wiki/%E5%B9%BF%E5%BA%A6%E4%BC%98%E5%85%88%E6%90%9C%E7%B4%A2)

- [深度优先搜索（需要梯子）](https://zh.wikipedia.org/wiki/%E6%B7%B1%E5%BA%A6%E4%BC%98%E5%85%88%E6%90%9C%E7%B4%A2)

- [深搜（DFS）与广搜（BFS）区别](https://www.cnblogs.com/wkfvawl/p/9347828.html)

- [一文搞懂深度优先搜索、广度优先搜索(dfs,bfs)](https://juejin.im/post/5d7127eb51882519a93ac19f)

- [Java实现深度优先遍历和广度优先遍历](https://blog.csdn.net/weixin_42289193/article/details/81741756)