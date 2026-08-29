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

#### Java 中的队列Queue（使用双端队列Deque代替）

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

> 为什么不能这样写 `for`？**问题**：`origin.poll();` 会不断删除元素，因此 `origin.size()` 也会不断变小，导致循环提前结束，但元素没有全部移动。因此，当循环过程中集合大小会发生变化时，更适合使用`while` 循环。

```java
for (int i = 0; i < origin.size(); i++) {
    temp.offer(origin.poll());
}
```


#### 复杂度

* `push()`：O(n)
* `pop()`：O(1)
* `top()`：O(1)
* `empty()`：O(1)

### 用栈实现队列 (LeetCode 232)

#### Java 中的栈（使用双端队列Deque代替）

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



### 设计循环队列（LeetCode 622）

#### 循环队列

循环队列（Circular Queue）是一种 **先进先出（FIFO）** 的队列结构。

它通常使用固定长度的数组实现，并将数组的首尾在逻辑上连接起来。当队尾到达数组末尾时，可以重新回到数组开头，继续使用已经空出来的位置。

例子：

假设循环队列容量为 `5`：

```text
初始：
[ ] [ ] [ ] [ ] [ ]

依次入队 1、2、3、4、5：

[1] [2] [3] [4] [5]
 |               |
front           rear
```

出队两个元素 `1、2`：

```text
[ ] [ ] [3] [4] [5]
         |       |
       front    rear
```

此时数组前面出现了空位。

继续入队 `6`、`7`，队尾到达末尾后会绕回数组开头：

```text
[6] [7] [3] [4] [5]
         |
       front
```

此时队列的实际顺序为：

```text
3 -> 4 -> 5 -> 6 -> 7
```

循环队列的主要优点是：

* 可以重复利用数组中已经空出的空间
* 避免普通数组队列前部空间无法再次使用的问题
* 入队和出队操作都可以达到 O(1)

通常使用 `front` 和 `rear` 分别记录队头和队尾的位置，并通过取模运算实现循环：

```java
index = (index + 1) % capacity;
```

当下标到达数组末尾后，下一次就会重新回到下标 `0`。

#### 解法

使用array实现循环队列，主要需要维护以下几个变量：

```java
int[] data;
int front; // 队头元素下标
int rear; // 队尾元素下标
int size; // 当前元素数量
int capacity; // 队列容量
```

初始化：

```java
front = 0;
rear = -1;
size = 0;
capacity = 5; // 例如，容量为5
```

##### 入队 enQueue

`% capacity` 保证 `rear` 到达数组末尾后重新回到下标 `0`：

```java
rear = (rear + 1) % capacity;
data[rear] = value;
size++;
```

##### 出队 deQueue

同样通过取模实现循环：

```java
front = (front + 1) % capacity;
size--;
```

##### 获取队头和队尾

```java
data[front]   // 队头
data[rear]    // 队尾
```

##### 判断空和满

```java
size == 0         // 空
size == capacity  // 满
```

#### 易错点

##### 1. `rear` 不能初始化为 0

错误：

```java
front = 0;
rear = 0;
```

入队时：

```java
rear = (rear + 1) % capacity;
```

第一次插入会直接放到 `data[1]`，而 `front` 仍指向 `data[0]`。

因此应初始化：

```java
rear = -1;
```

第一次入队后：

```java
rear = (-1 + 1) % capacity; // 0
```

##### 2. 必须使用取模实现循环

错误：

```java
rear++;
front++;
```

下标到达数组末尾后会越界。

正确：

```java
rear = (rear + 1) % capacity;
front = (front + 1) % capacity;
```

##### 3. 不需要手动判断 rear 和 front 的位置关系

不需要写：

```java
if (rear > front ...)
if (rear < front ...)
```

循环队列的下标移动可以统一交给：

```java
(index + 1) % capacity
```

### 滑动窗口最大值（LeetCode 239）


#### Java 中的双端队列 Deque

> 双端队列也可以使用Stack和Queue的指令：

```java
// Stack
deque.push(1);   // 从队头加入
deque.pop();     // 从队头删除
deque.peek();    // 查看队头

// Queue
deque.offer(1);  // 从队尾加入
deque.poll();    // 从队头删除
deque.peek();    // 查看队头
```

双端队列操作：

```java
import java.util.Deque;
import java.util.ArrayDeque;

Deque<Integer> deque = new ArrayDeque<>();

deque.offerFirst(1);   // 从队头加入
deque.offerLast(2);    // 从队尾加入
deque.offer(3);        // 从队尾加入

deque.pollFirst();     // 删除并返回队头
deque.pollLast();      // 删除并返回队尾

deque.peekFirst();     // 查看队头，不删除
deque.peekLast();      // 查看队尾，不删除

deque.isEmpty();       // 是否为空
deque.size();          // 元素个数
```

#### 解法

##### 暴力解法 (O(nk))

```java
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        int length = nums.length - k + 1;
        int[] ans = new int[length];
        
        for (int i = 0; i < length; i++) {
            double local_max = Double.NEGATIVE_INFINITY;
            // 注意这里是 j < i + k, 而不是j < length
            for (int j = i; j < i + k; j++) {
                if (nums[j] > local_max) {
                    local_max = nums[j];
                }
            }
            ans[i] = (int) local_max;
        }

        return ans;
    }
}
```

##### 双端队列解法 (O(n))

```java
import java.util.Deque;
import java.util.ArrayDeque;

class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        int[] ans = new int[nums.length - k + 1];
        Deque<Integer> deque = new ArrayDeque<>();

        int index = 0;

        for (int i = 0; i < nums.length; i++) {

            // 1. 删除已经离开窗口的元素
            if (!deque.isEmpty() && deque.peekFirst() <= i - k) {
                deque.pollFirst();
            }

            // 2. 保持队列单调递减
            while (!deque.isEmpty()
                    && nums[deque.peekLast()] < nums[i]) {
                deque.pollLast();
            }

            // 保存下标
            deque.offerLast(i);

            // 3. 窗口形成后，队头就是最大值
            if (i >= k - 1) {
                ans[index++] = nums[deque.peekFirst()];
            }
        }

        return ans;
    }
}
```

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

