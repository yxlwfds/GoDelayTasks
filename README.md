## 延时任务队列

### 思路发展历程

#### 第一阶段   单级时间轮

一个有60刻度的时间轮，每个tick一秒钟，主线程持续走动timer，当有新的任务到来交给工作线程创建。

**优点**

实现简单

**缺点**

1. 当同一时刻有多个任务时无法插入
2. 可承载任务过少只有60个

#### 第二阶段   单级双向链表时间轮

一个有3600刻度的时间轮，每个tick表示一秒钟，最多延时一个小时，每个时间槽为一个 **双向循环链表**，这样可以在同一时刻保存多个任务。

**优点**

1. 实现了同一时刻多个任务的保存
2. 扩大了可承载任务到1小时

**缺点**

1. 时间跨度还是太小只有一小时
2. 时间精度太低

#### 第三阶段   多级时间轮

和时间的表示一样，按照天、小时、分、秒、毫秒分为五个时间轮，这样可表示的跨度理论上到了一年（某天某小时某分钟某秒某毫秒），每个tick为一毫秒，同样每个时间槽都是双向循环链表。

**优点**

1. 跨度变大，足够使用
2. 时间精度够高，对于高精度的延时任务有保障

**缺点**

1. 实现较为复杂，对于每一层时间轮的过期任务需要往后移动，每一级时间轮之间的依赖性太强
2. 依赖服务器设置时间，如果修改了系统时间后所有任务受影响

#### 第四阶段   

主要的改变（**参考Netty的时间轮实现**），利用位的概念设计时间（虽然是参考标准时间小时、分钟、秒这样的实现，但是不要被这个概念框限住）。根据一个时间来分时间轮级别，我们将一个时间表示为32位的时间，分为

1. 一级时间轮，低8位，表示0～256毫秒的延时任务，tick单位是一毫秒
2. 二级时间轮，次6位，表示256～256*64毫秒的延时任务，tick单位是256毫秒，也就是每256毫秒走动一次
3. 三级时间轮，低第二个6位，表示256～256 * 64 * 64毫秒的延时任务，tick单位是256 * 64毫秒，也就是每64 * 256毫秒走动一次
4. 四级时间轮，低第三个6位，表示256～256 * 64 * 64 * 64毫秒的延时任务，tick单位是256 * 64毫秒，也就是每64 * 64 * 256毫秒走动一次
5. 五级时间轮，低第四个6位，表示256～256 * 64 * 64 * 64 * 64毫秒的延时任务，tick单位是256 * 64 * 64 * 64毫秒，也就是每64 * 256毫秒走动一次

这样最大的好处就是每个时间段的延时任务分到了五个不同时间轮，并且可以使用位运算进行时间槽流转，相对于加减乘除运算速度也更快

并且计时系统使用的是 **monotonic** 时间，它表示从系统启动这一刻起到现在的时长。不受系统时间被用户修改的影响。如果系统不支持 monotonic 时间的话，就可能出现用户修改系统时间，导致计时器混乱的情况。如果出现这种情况，需要进行时间校正。代码中使用了 monotonic 时间，无需时间校正。
