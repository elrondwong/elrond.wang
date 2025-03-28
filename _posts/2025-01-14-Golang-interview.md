# 知识点

## goroutine为什么比线程快

- 创建调度与销毁: goroutine的创建调度与销毁都在用户态，线程的在内核态
- 内存分配: goroutine栈内存为2-4KB，线程1MB
- 切换: goroutine切换时仅需要保护少数寄存器，而线程则需要保护大量的寄存器，goroutine无需操作系统线程切换

## GMP

- 两级线程模型，内核调度实体kse与线程的对应关系为m:n
- kse -> m -> p ->g
  - m: machine 关联一个内核线程
  - p: processor 作为上下文，衔接m和g
  - g: goroutine 轻量化线程，调用栈、调用信息， 例如channel等

<https://github.com/fengyuan-liang/notes/blob/main/GoLang/golang%E5%A4%A7%E6%9D%80%E5%99%A8GMP%E6%A8%A1%E5%9E%8B.md>

<https://www.jianshu.com/p/4afa0679851d>

## slice的扩容

golang 1.21.4

- 对于小slice进行1.25倍扩展 逐渐过度
- 对于大slice进行2倍扩展

src/runtime/slice.go

```golang
 newcap := oldCap
 doublecap := newcap + newcap
 if newLen > doublecap {
  newcap = newLen
 } else {
  const threshold = 256
  if oldCap < threshold {
   newcap = doublecap
  } else {
   // Check 0 < newcap to detect overflow
   // and prevent an infinite loop.
   for 0 < newcap && newcap < newLen {
    // Transition from growing 2x for small slices
    // to growing 1.25x for large slices. This formula
    // gives a smooth-ish transition between the two.
    newcap += (newcap + 3*threshold) / 4
   }
   // Set newcap to the requested cap when
   // the newcap calculation overflowed.
   if newcap <= 0 {
    newcap = newLen
   }
  }
 }
```

## g0

持有调度栈的goroutine，作用有

- goroutine的创建
- 大内存分配
- CGO函数的执行

## 三色标记法

<https://community.apinto.com/d/34057-golang-gc>

# Data Structures and

## Sliding Window(滑动窗口)

第三题

<https://leetcode.cn/problems/longest-substring-without-repeating-characters/>

关键点:

- index>=left: index一定需要在left右边，保证取值不会超过当前窗口
- left=index+1: 如果前面的有重复了说明当前窗口已经是最大状态，从index+1开始才能保证和当前窗口无重复字母
- index的更新: 有重复字母则更新对应的坐标

```golang
func lengthOfLongestSubstring(s string) int {
  charMap := make(map[byte]int)
  maxLength := 0
  left, right := 0,0
  for right=0;right < len(s);right++ {
    if index, ok := charMap[s[right]]; ok && index >= left {
      left = index + 1
    }
    charMap[s[right]] = right
    maxLength = max(maxLength, right-left+1)
  }
  return maxLength
}
```

## Brute Force(暴力枚举)

## 递归（Recursion）

## 回溯（Backtracking）

## 分治（Divide and Conquer）

## 动态规划（Dynamic Programming, DP）

第五题，最大回文子串

https://leetcode.cn/problems/longest-palindromic-substring/description/

关键点:

- 先算小的，小的算完之后记录结果, 用小的结果计算更大的记录结果，然后得到大的结果
  - 单个字符一定是回文 dp[i][i]=true
  - 相邻字符如果是相同字符一定是回文 dp[i][i+1]=true
  - 大于三个的则可以通过 dp[i][j] 如果头尾一样 s[i] == s[j], 那么判断 dp[i+1][j-1] 为true则为true
  - 那么通过从小到多的遍历就能获取到所欲呕的dp[i][j]
  - 第一次遍历, 处理只有一位的回文
  - 第二次遍历，填充二位回文
  - 第三次遍历，填充3位及以上回文，这里需要注意
    - 由于前两次遍历已处理完长度为1和长度为2的回文字符串，所以遍历时字符串长度从3开始，进行遍历所有大于3的长度的字符串填充
    - 有了字符串长度的一重循环，内部需要从第一个字符开始遍历，最大遍历到字符串总体长度-当前计算的字符串长度，注意边界问题
      - 例如一个字符串长度为9，当前计算的长度为5 那么最后一个坐标应该是 9-5=4，可以取到4吗 0 1 2 3 4 5 6 7 8， 取到4 0 1 2 3 + 后面五位 4 5 6 7 8 9 所以这里可以取到这个边界值
      - 二重循环是有了起点 还需要有一个终点， 这个终点就是 起点坐标+长度-1 保证不会超出整个字符串的长度

```golang
func longestPalindrome(s string) string {
 n := len(s)

 if n == 0 {
  return ""
 }

 dp := make([][]bool, n)
 for i := range dp {
  dp[i] = make([]bool, n)
 }
 start, maxLen := 0, 1
 for i := 0; i < n; i++ {
  dp[i][i] = true
 }
 for i := 0; i < n-1; i++ {
  if s[i] == s[i+1] {
   dp[i][i+1] = true
   start = i
   maxLen = 2
  }
 }

 for length := 3; length <= n; length++ {
  for i := 0; i <= n-length; i++ {
   j := i + length - 1
   if s[i] == s[j] && dp[i+1][j-1] {
    dp[i][j] = true
    start = i
    maxLen = length
   }
  }
 }
 return s[start : start+maxLen]

```

## 贪心算法（Greedy Algorithm）

## 双指针（Two Pointers）

## 并查集（Union-Find）

## 拓扑排序（Topological Sorting）

## 图算法（Graph Algorithms）

## 位运算（Bit Manipulation）

## 单调栈 & 单调队列（Monotonic Stack & Queue）

# Algorithms

## LRU

第146题

<https://leetcode.cn/problems/lru-cache/submissions/609910341/>

```golang
package lrucache

type LRUCache struct {
 cache            map[int]*Node
 capacity, length int
 head, tail       *Node
}

type Node struct {
 key, value int
 next, prev *Node
}

func Constructor(capacity int) LRUCache {
 head, tail := &Node{}, &Node{}
 head.next = tail
 tail.prev = head
 return LRUCache{
  cache:    make(map[int]*Node, capacity),
  capacity: capacity,
  head:     head,
  tail:     tail,
 }
}

func (this *LRUCache) Get(key int) int {
 if node, ok := this.cache[key]; ok {
  this.movetoHead(node)
  return node.value
 }
 return -1

}

func (this *LRUCache) Put(key int, value int) {
 node, ok := this.cache[key]
 if ok {
  node.value = value
  this.movetoHead(node)
 } else {
  if this.length >= this.capacity {
   this.removeTail()
  }
  node = &Node{key: key, value: value}
  this.addToHead(node)
  this.cache[key] = node
  this.length++
 }
}

func (this *LRUCache) addToHead(node *Node) {
 this.head.next.prev = node
 node.next = this.head.next
 this.head.next = node
 node.prev = this.head

}

func (this *LRUCache) removeTail() {
 if this.tail.prev == this.head {
  return
 }
 node := this.tail.prev
 this.removeNode(node)
 delete(this.cache, node.key)
 this.length--
}

func (this *LRUCache) movetoHead(node *Node) {
 this.removeNode(node)
 this.addToHead(node)
}

func (this *LRUCache) removeNode(node *Node) {
 node.prev.next = node.next
 node.next.prev = node.prev
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * obj := Constructor(capacity);
 * param_1 := obj.Get(key);
 * obj.Put(key,value);
 */

```
