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

### [回溯] 复原IP地址（LeetCode 93）

#### String

* `length()`：长度
* `charAt(i)`：取第 `i` 个字符
* `equals(s)`：比较字符串内容
* `contains(s)`：是否包含
* `indexOf(s)`：查找位置，找不到返回 `-1`
* `substring(a, b)`：截取 `[a, b)`
* `replace(a, b)`：替换
* `split(s)`：切割字符串
* `toUpperCase()` / `toLowerCase()`：大小写转换
* `trim()`：去除首尾空格
* `startsWith(s)` / `endsWith(s)`：判断开头/结尾
* `isEmpty()`：是否为空字符串
* `Integer.parseInt(s)`：将字符串转换为整数
* `Double.parseDouble(s)`：将字符串转换为双精度浮点数

```java
String s = "Hello Java";

s.length();              // 10
s.charAt(0);             // 'H'
s.equals("Hello Java");  // true
s.contains("Java");      // true
s.indexOf("Java");       // 6
s.substring(6);          // "Java"
s.substring(0, 5);       // "Hello"
s.replace("Java", "World");
s.split(" ");
```

> `String` 不可变；字符串内容比较用 `equals()`，不要用 `==`。


> `String` 不可变，不能直接插入字符（插入字符：`s.substring(0, i) + c + s.substring(i)`；频繁修改用 `StringBuilder`）：

```java
String s = "helo";
s = s.substring(0, 3) + 'l' + s.substring(3); // "hello"

StringBuilder sb = new StringBuilder("helo");
sb.insert(3, 'l'); // "hello"
```

#### ArrayList

不知道最后会有多少元素，通常用 ArrayList

```java
import java.util.ArrayList;
ArrayList<String> list = new ArrayList<>();
list.add(x);      // 加到最后
list.get(i);      // 取第 i 个
list.set(i, x);   // 修改第 i 个
list.remove(i);   // 删除第 i 个
list.size();      // 长度
```

#### 解法

**思路：** 枚举 3 个分割点，把字符串切成 4 段，每段检查是否合法。

```java
class Solution {
    public List<String> restoreIpAddresses(String s) {
        List<String> ans = new ArrayList<>();

        for (int i = 1; i <= 3 && i < s.length(); i++) {
            for (int j = i + 1; j <= i + 3 && j < s.length(); j++) {
                for (int k = j + 1; k <= j + 3 && k < s.length(); k++) {

                    String a = s.substring(0, i);
                    String b = s.substring(i, j);
                    String c = s.substring(j, k);
                    String d = s.substring(k);

                    if (valid(a) && valid(b) && valid(c) && valid(d)) {
                        ans.add(a + "." + b + "." + c + "." + d);
                    }
                }
            }
        }
        return ans;
    }

    private boolean valid(String s) {
        if (s.length() == 0 || s.length() > 3) return false;
        if (s.length() > 1 && s.charAt(0) == '0') return false;
        return Integer.parseInt(s) <= 255;
    }
}
```

**注意：**
* 记得判断`"0"` 合法，`"01"` 不合法
* `substring(a, b)` 为 `[a, b)`（分割点不能相同，否则会产生 `""`）
* 截取时候不要超过字符串长度

**复杂度：** 最多 `3³ = 27` 种分割，≈ `O(1)`。

#### Backtracking（回溯法）

回溯法就是把loop改成递归，在input长度不可控时，可以实现 pruning branch 效果，优化runtime。

```java
class Solution {
    public List<String> restoreIpAddresses(String s) {
        List<String> result = new ArrayList<>();
        List<String> path = new ArrayList<>();

        backtrack(s, 0, path, result);

        return result;
    }

    private void backtrack(String s, int start, List<String> path, List<String> result) {
        // 递归终止条件1：已经选了 4 段
        if (path.size() == 4) {
            // 递归终止条件2：刚好用完整个字符串
            if (start == s.length()) {
                result.add(String.join(".", path));
            }
            return;
        }

        // 当前这一段尝试长度 1-3
        for (int len = 1; len <= 3; len++) {

            int end = start + len;
            if (end > s.length()) break; // 检查：超出字符串范围

            String part = s.substring(start, end);

            // 当前段不合法
            if (!isValid(part)) continue;

            // 1. 选择
            path.add(part);

            // 2. 继续选下一段
            backtrack(s, end, path, result);

            // 3. 撤销选择
            path.remove(path.size() - 1);
        }
    }

    private boolean isValid(String part) {
        // 不能有前导 0
        if (part.length() > 1 && part.charAt(0) == '0') {
            return false;
        }

        // 必须 <= 255
        return Integer.parseInt(part) <= 255;
    }
}
```


### [回溯] 非递减子序列（LeetCode 491）

最典型的回溯算法结构，必看！

> 核心：Backtracking 枚举子序列 + `path` 最后一个元素保证非递减 + `Set` 同层去重 + 每次递归结束撤销 `path` 以恢复上一层状态。

#### Set / HashSet

`Set` 用来保存**不重复元素**，这题用于 Backtracking 的**同层去重**。

```java
import java.util.Set;
import java.util.HashSet;

Set<Integer> used = new HashSet<>();

used.add(x);        // 添加元素
used.contains(x);   // 是否已经存在 x
used.remove(x);     // 删除 x
used.size();        // 元素数量
used.isEmpty();     // 是否为空
```

> `Set` 不允许重复元素。这题中每一层递归都创建一个新的 `used`，只记录“当前这一层已经选择过哪些数字”。

#### List 相关操作

```java
path.isEmpty();                  // 是否为空
path.get(i);                     // 获取第 i 个元素
path.get(path.size() - 1);       // 获取最后一个元素
path.add(x);                     // 添加元素
path.remove(path.size() - 1);    // 删除最后一个元素
new ArrayList<>(path);           // 复制当前 List
```

> 保存答案时必须使用 `new ArrayList<>(path)`，因为 `path` 在后续递归中还会继续被修改。

#### 解法：Backtracking

**目标：** 找出所有长度 `>= 2` 的非递减子序列。

1. 为什么使用 Backtracking

    每个位置都有“选 / 不选”的可能，而且选了一个数字后，还需要继续尝试后面的数字，因此需要枚举很多不同的子序列。

    Backtracking 的基本结构：

    ```java
    做选择;
    backtrack(...);
    撤销选择;
    ```

    这题中对应：

    ```java
    path.add(nums[i]);                 // 选择当前数字
    backtrack(nums, i + 1, path, ans); // 继续搜索后面的数字
    path.remove(path.size() - 1);      // 撤销当前选择
    ```

2. `path`：保存当前子序列

    ```java
    List<Integer> path = new ArrayList<>();
    ```

    例如：

    ```text
    path = [4, 6]
    ```

    表示当前已经选择了 `4 -> 6`。

    如果继续选择 `7`：

    ```java
    path.add(7);
    ```

    则：

    ```text
    path = [4, 6, 7]
    ```

3. `start`：保证子序列保持原数组顺序

    每次选择 `nums[i]` 后，下一层只能从 `i + 1` 开始：

    ```java
    backtrack(nums, i + 1, path, ans);
    ```

    例如：

    ```text
    nums = [4, 6, 7, 7]
            0  1  2  3
    ```

    选择 index `1` 的 `6` 后，只能继续搜索 index `2`、`3`，不能回头再选择 `4`。

    > 因此 `start` 保证结果是“子序列”，而不是任意重新排列。

4. 保证非递减

    新加入的数字必须满足：

    ```text
    nums[i] >= path 最后一个数字
    ```

    判断：

    ```java
    if (!path.isEmpty()
            && nums[i] < path.get(path.size() - 1)) {
        continue;
    }
    ```

    例如：

    ```text
    path = [4, 7]
    当前 nums[i] = 6

    6 < 7 -> 不能加入 -> continue
    ```

5. 长度 >= 2 就记录答案

    ```java
    if (path.size() >= 2) {
        ans.add(new ArrayList<>(path));
    }
    ```

    不需要等到走到数组末尾，因为：

    ```text
    [4,6]
    ```

    本身已经是一个合法答案，即使后面还能继续变成：

    ```text
    [4,6,7]
    [4,6,7,7]
    ```

    保存答案时不能写：

    ```java
    ans.add(path); // 错误
    ```

    因为 `path` 是同一个 List，后面会继续 `add()` / `remove()`。

    正确：

    ```java
    ans.add(new ArrayList<>(path));
    ```

    相当于把当前状态复制一份保存下来。

6. 同层去重（重要）

    例如：

    ```text
    nums = [4, 6, 7, 7]
                |  |
    ```

    当：

    ```text
    path = [4,6]
    ```

    这一层如果选择第一个 `7`：

    ```text
    [4,6,7]
    ```

    再选择第二个 `7`：

    ```text
    [4,6,7]
    ```

    会产生完全相同的结果。

    所以同一层中，同一个数字只能作为一次选择：

    ```java
    Set<Integer> used = new HashSet<>();

    if (used.contains(nums[i])) {
        continue;
    }

    used.add(nums[i]);
    ```

    > **同层去重：** `used` 只限制同一个父节点下的选择。例如当前 `path = [4,6]` 时，如果这一层有两个 `7`，无论选哪一个都会得到 `[4,6,7]`，所以第二个 `7` 应该跳过；但 `[4] -> [4,7] -> [4,7,7]` 是合法的，因为两个 `7` 位于不同递归层。因此 `used` 必须定义在 `backtrack()` 内部，使每一层递归都有自己独立的 `used`。同时，`used` 不需要在回溯时 `remove()`，因为它记录的是“这一层已经尝试过哪些数字”，即使某个分支已经搜索结束，这一层也不应该再次选择相同的数字。

8. 撤销选择（重要）

    这是 Backtracking 最重要的部分。

    假设当前：

    ```text
    path = [4, 6]
    ```

    选择 `7`：

    ```java
    path.add(7);
    ```

    此时：

    ```text
    path = [4, 6, 7]
    ```

    然后进入递归：

    ```java
    backtrack(...);
    ```

    这一条以 `[4,6,7]` 开头的路线全部搜索完后，需要继续尝试 `[4,6]` 的其他选择。

    所以必须：

    ```java
    path.remove(path.size() - 1);
    ```

    把状态恢复为：

    ```text
    [4, 6]
    ```

    然后才能从 `[4,6]` 出发尝试另一条分支。

    > **回溯 = 递归搜索完一个分支后，把共享状态恢复到进入这个分支之前。**

#### 搜索过程示例

`nums = [4, 6, 7]`

大致过程：

```text
path = []

选择 4
path = [4]
    选择 6
    path = [4,6]       -> 记录
        选择 7
        path = [4,6,7] -> 记录
        撤销 7
        path = [4,6]
    撤销 6
    path = [4]

    选择 7
    path = [4,7]       -> 记录
    撤销 7
    path = [4]

撤销 4
path = []

选择 6
path = [6]
    选择 7
    path = [6,7]       -> 记录
    撤销 7

撤销 6
...
```

这就是“选择 -> 递归 -> 撤销 -> 尝试下一个选择”。

#### 完整代码

```java
import java.util.*;

class Solution {
    public List<List<Integer>> findSubsequences(int[] nums) {
        List<List<Integer>> ans = new ArrayList<>();
        List<Integer> path = new ArrayList<>();

        backtrack(nums, 0, path, ans);

        return ans;
    }

    private void backtrack(
        int[] nums,
        int start,
        List<Integer> path,
        List<List<Integer>> ans
    ) {
        // 当前 path 已经是合法子序列
        if (path.size() >= 2) {
            ans.add(new ArrayList<>(path));
        }

        // 当前递归层已经使用过的数字
        Set<Integer> used = new HashSet<>();

        for (int i = start; i < nums.length; i++) {

            // 条件 1：保证非递减
            if (!path.isEmpty()
                    && nums[i] < path.get(path.size() - 1)) {
                continue;
            }

            // 条件 2：同层去重
            if (used.contains(nums[i])) {
                continue;
            }

            // 标记：这一层已经选择过 nums[i]
            used.add(nums[i]);

            // 1. 做选择
            path.add(nums[i]);

            // 2. 递归：从后一个位置继续搜索
            backtrack(nums, i + 1, path, ans);

            // 3. 撤销选择：恢复进入该分支前的 path
            path.remove(path.size() - 1);
        }
    }
}
```

#### 易错点

* `start = i + 1`：保证子序列保持原数组顺序。
* 数组不能先排序，否则会改变原数组顺序，得到原本不存在的子序列。
* `path.size() >= 2` 就要记录答案，不需要等到数组遍历结束。
* `ans.add(new ArrayList<>(path))`：必须复制当前 `path`。
* `path.add()` 后递归结束必须 `path.remove()`，否则会污染下一条分支。
* `used` 是**同层去重**，不是整个递归过程去重。
* `used` 不需要回溯删除，因为同一层已经尝试过的值不能再次尝试。
* `path` 需要回溯删除，因为它代表当前递归路径，返回上一层后必须恢复原状态。

#### 复杂度

- 时间复杂度：O(2^n)
- 空间复杂度：O(n) 


### [贪心] 发饼干（LeetCode 455）

#### 数组排序

```java
import java.util.Arrays;
int[] nums = {3, 1, 4, 2};
Arrays.sort(nums); // 直接修改原数组
```

#### 解法

**思路：** 排序 + 双指针。用**最小但能满足当前孩子的饼干**，避免浪费大饼干。


```java
import java.util.Arrays;

class Solution {
    public int findContentChildren(int[] g, int[] s) {
        Arrays.sort(g);
        Arrays.sort(s);

        int i = 0; // child
        int j = 0; // cookie
        int count = 0;

        while (i < g.length && j < s.length) {
            if (s[j] >= g[i]) {
                count++;
                i++;
                j++;
            } else {
                j++;
            }
        }

        return count;
    }
}
```

**注意：** 不能双重 loop（比如for loop嵌套） 直接 `count++`，否则会重复使用孩子/饼干

#### 复杂度

* 时间：`O(n log n + m log m)`
* 空间：`O(1)`（不考虑排序内部空间）



### [贪心] 柠檬水找零（LeetCode 860）

#### 解法

**思路：** 只需要记录 `$5` 和 `$10` 的数量。收到 `$20` 时优先找 `$10 + $5`，尽量保留更多 `$5`。

```java
class Solution {
    public boolean lemonadeChange(int[] bills) {
        int five = 0;
        int ten = 0;

        for (int bill : bills) {
            switch (bill) {
                case 5:
                    five++;
                    break;

                case 10:
                    if (five == 0) return false;
                    five--;
                    ten++;
                    break;

                case 20:
                    if (ten > 0 && five > 0) {
                        ten--;
                        five--;
                    } else if (five >= 3) {
                        five -= 3;
                    } else {
                        return false;
                    }
                    break;
            }
        }

        return true;
    }
}
```

**注意：**

* `$20` 找零优先 `$10 + $5`，否则 `$5 + $5 + $5`
* `$5` 最重要，因为 `$10`、`$20` 找零都可能需要
* 不需要记录 `$20`，因为不会拿它找零
* **`switch case` 记得写 `break`，否则会继续执行下一个 `case`**

#### 复杂度

* 时间：`O(n)`
* 空间：`O(1)`









