### HashMap原理概述

- 在1.7中，HashMap内部通过数组+链表的形式实现。数组的初始长度是16，会在第一次put时初始化；满载率是0.75，也就是说当元素达到16*0.75=12个后就会扩容。扩容时每次增长后的长度为2的整数次幂。

- HashMap在put时会判断是否需要扩容，需要扩容的话会通过reSize()进行扩容，会新建一个数组，然后会通过indexFor()找到元素位置，再把元素一一拷贝过去。不需要扩容的话会把元素包装成一个对象。通过二次hash来减小hash碰撞，并确定数据在数组中的位置。如果二次hash后不同的元素hash后的结果仍然在同一位置。那么通过链表的方式存储，这种方式叫做拉链法。存储时后存入的元素总在头部，这种方式叫做头插法，这么设计的原因是一般认为后插入的元素更容易被访问到。

- HashMap在get时会通过hash直接确定元素在数组中的位置，这样效率更高。如果这个位置存在链表，再通过遍历链表的方式找到元素。

- 1.8后HashMap增加了红黑树，当数组长度大于等于64，链表超过一定长度（默认为8）时，改为红黑树结构， 1.8采用了尾插法。

- 1.8后HashMap扩容时通过⾼低位的桶直接在链表尾部添加。

  

