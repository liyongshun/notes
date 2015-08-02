#CleanCode#
##1. 为什么要CleanCode？##

“Clean Code That Works”，来自于*Ron Jeffries*这句箴言指导我们写的代码要整洁有效，*Kent Beck*把它作为TDD（Test Driven Development）追求的目标，*BoB大叔*（Robert C. Martin）甚至写了一本书来阐述他的理解。
你可能不这么认为：**Works**不就Ok了么？为什么还要**CleanCode**呢?
*kent Beck*给出了如下原因：
+ It is a predictable way to develop. You know when you are finished, without having to worry about a long bug trail.
+ It gives you a chance to learn all of the lessons that the code has to teach you. If you only slap together the first thing you think of, then you never have time to think of a second, better thing.
+ It improves the lives of the users of your software.
+ It lets your teammates count on you, and you on them.
+ It feels good to write it.

我们分析他的观点，主要如下几个要点：
 + 整洁的代码质量更好，故障就像白墙上的一只苍蝇，如此的显眼，往往令其无处藏身。开发的高质量，缩短验证的周期，让交付更容易被估算。
 + 最容易想到的方法往往是糟糕的方法，*CleanCode*的代码需要花费心思去推敲，更容易抓住业务的本质，一次性把事情做对。
 + 整洁的代码可读性更好，风格统一，开发小组之间更容易产生信任关系

##2. 如何做到CleanCode？##
###简单设计四原则：###
> 
+ Passes the tests
+ Reveals intention
+ No duplication
+ Fewest elements

优先级如下：
![SimpleDesignPrinciple][simple-design-principle]

###测试驱动开发###

![Tdd][tdd]





![Mind map][clean-code]

[tdd]: images/clean-code/tdd.png
[simple-design-principle]: images/clean-code/SimpleDesignPrinciple.png
[clean-code]: images/clean-code/cleanCode.png "CleanCode MindMap"



