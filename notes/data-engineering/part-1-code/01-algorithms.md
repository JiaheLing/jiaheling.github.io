---
title: "算法基础"
---

---
## 数组



---
## 链表

---
## 字符串

---
## 队列与栈

### 用队列实现栈 (LeetCode 225)

#### Java 中的队列 Queue

队列（Queue）是一种 **先进先出（FIFO）** 的数据结构。

Java 中常用 `Queue` 接口表示队列：

```java
import java.util.Queue;
import java.util.ArrayDeque;

Queue<Integer> queue = new ArrayDeque<>();
```

常用操作：

```java
queue.offer(1);      // 入队
queue.poll();        // 出队，删除并返回队头元素
queue.peek();        // 查看队头元素
queue.isEmpty();     // 判断队列是否为空
queue.size();        // 获取队列长度
```

例如：

```java
queue.offer(1);
queue.offer(2);
queue.offer(3);
```

队列中的顺序为：

```text
1, 2, 3
```

执行：

```java
queue.poll();
```

会取出 `1`。

#### 解法

思路：始终让 `origin` 的队头保存最新加入的元素。

1. 将 `origin` 中的所有元素移动到 `temp`
2. 将新元素 `x` 放入 `origin`
3. 再将 `temp` 中的元素按原顺序移回 `origin`

例如：

```text
origin = [3, 2, 1]

push(4)

temp   = [3, 2, 1]
origin = [4]

移动回来后：

origin = [4, 3, 2, 1]
```

这样，最新加入的元素始终位于队头，所以：

```java
pop() -> origin.poll()
top() -> origin.peek()
```

代码：

```java
public void push(int x) {
    temp = new ArrayDeque<>();

    while (!origin.isEmpty()) {
        temp.offer(origin.poll());
    }

    origin.offer(x);

    while (!temp.isEmpty()) {
        origin.offer(temp.poll());
    }
}
```

> 为什么不能这样写 `for`？

```java
for (int i = 0; i < origin.size(); i++) {
    temp.offer(origin.poll());
}
```

    问题：`origin.poll();` 会不断删除元素，因此 `origin.size()` 也会不断变小，导致循环提前结束，但元素没有全部移动。
    
    因此，当循环过程中集合大小会发生变化时，更适合使用`while` 循环。

#### 复杂度

* `push()`：O(n)
* `pop()`：O(1)
* `top()`：O(1)
* `empty()`：O(1)

### 用栈实现队列 (LeetCode 232)

#### Java 中的栈

栈（Stack）是一种 **后进先出（LIFO）** 的数据结构。

Java 中常用 `Deque` 接口来表示栈，通常使用 `ArrayDeque` 作为实现：

```java
import java.util.Deque;
import java.util.ArrayDeque;

Deque<Integer> stack = new ArrayDeque<>();
```

常用操作：

```java
stack.push(x);       // 入栈（将元素 x 压入栈顶）（队列用offer()）
stack.pop();         // 出栈，删除并返回栈顶元素（队列用poll()）
stack.peek();        // 查看栈顶元素，但不删除
stack.isEmpty();     // 判断栈是否为空
stack.size();        // 获取栈的大小
```

#### 解法

思路与"用两个队列实现栈"类似：在 `push()` 时调整元素顺序，使 `origin` 的栈顶始终保存最早进入的元素。`push(x)` 时：

1. 将 `origin` 中的所有元素移动到 `temp`
2. 将新元素 `x` 压入 `origin`
3. 再将 `temp` 中的元素移回 `origin`

```java
public void push(int x) {
    temp = new ArrayDeque<>();

    while (!origin.isEmpty()) {
        temp.push(origin.pop());
    }

    origin.push(x);

    while (!temp.isEmpty()) {
        origin.push(temp.pop());
    }
}
```

因此：

```java
pop()  -> origin.pop()
peek() -> origin.peek()
```

#### 复杂度：

* `push()`：O(n)
* `pop()`：O(1)
* `peek()`：O(1)
* `empty()`：O(1)





---
## 树与堆

---
## 散列表

---
## 图

---
## 排序

---
## 递归

---
## 分治

---
## 动态规划

---
## 回溯与贪心

