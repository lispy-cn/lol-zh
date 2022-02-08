# 附录 B：不推荐的语言
---

Author: Doug Hoyte <[doug@hoytech.com](mailto:doug@hoytech.com)>

Translator: Yuqi Liu <[yuqi.lyle@outlook.com](mailto:yuqi.lyle@outlook.com)>

---

## B.1 个人观点


> There are only two industries that refer to their
> customers as "users".
> -- Edward Tufte

 本附录中的所有内容，甚至整本书，都必须被视为我的意见。 然而，我个人并不认为这个附录是种个人意见，而是从 lisp 的角度对一些非 lisp 语言哲学及其问题的简要调查。 并不是说用这些语言编写好的代码是不可能的，也不是说没有聪明的人会用这些语言编程和工作。 对此，我无意冒犯，请多多包含。 这并不是针对你们的价值观。



## B.2 Blub Central


Blub 语言的出现时间甚至比 lisp 的出现时间还要长。 在 lisp 语言标准化并编写出体面的 lisp 编译器之前，通常确实需要易于实现但对正规编程很差的 Blub 语言。 但是，既然我们的软件开发能力如此之大，为什么大多数程序员使用少数几种语言，无论出于何种意图和目的，难道他们都无法进行区分？ 我认为这可以用博弈论来解释。


就像汽车经销店、咖啡馆和大学一样，通常会出现明显的集群效应，其中机构似乎不合理地靠得很近。 如果所有这些机构都均匀分布而不是聚集在一起，那么它们很可能服务于更大的市场，从而各自获得更大的个人利润。 但是因为这些机构无法合作，而且如果任何一个机构搬到遥远的地方，情况会比不这样做更糟糕，没有人移动，集聚仍然存在。


编程语言中也有类似的现象——我称之为 Blub Central。 大多数主要编程语言都无法或不愿意提供新功能，甚至无法改进其基本设计，因为它们忙于尝试像其他主要编程语言一样。 如果编程语言的设计者和开发者能够合作，结果将是一个更加多样化、功能更加丰富的语言家族，并且找到一种完全适合你的问题的语言的机会将会增加。 当然，大多数编程语言供应商要么是大公司，要么是很少合作的学术委员会，所以就有了目前的情况：大多数程序员都被硬塞进了 Blub Central。 因为几乎所有程序员都学习这些 Blub Central 语言，所以从 Blub Central 转移任何一种语言都意味着失去市场份额。


尽管对于 lisp 程序员来说听起来很奇怪，但对于 Blub Central 语言似乎有合理的经济解释。 全世界的程序员资源被浪费在每项工作上都使用相同的蹩脚工具令人震惊和沮丧。 但幸运的是，对于世界各地的程序员来说，有一种快速、不间断、单程的离开 Blub Central 的旅程：Common Lisp 快车。


## B.3 Niche Blub


实际上有数千种编程语言，并不是所有的语言都可以竞争 Blub Central。 当然，还有 Niche Blubs。 其中一些甚至非常接近提供 lisp 风格的功能集，尽管没有一个拥有像 lisp 这样的宏系统，这让他们和 Blub 区别开来。


这些语言中的许多都因缺乏可信赖的、稳定的标准以及被误导的哲学而受苦。 通常它们是围绕一个难以捉摸的、没有明确规定的目标设计的——良好风格的目标。 没有人真正确定这些语言应该去哪里或应该如何改变。 设计师可以并且确实决定将语言更改为被认为具有更好风格的语言，然后其他所有人都必须四处更改他们的代码以符合要求。 应该避免强加一种真正的编程风格而不尊重用户的语言。


在 lisp 中，几乎总是有一条客观正确的路要走：如果它使语言更强大，就应该添加它。 如果它使语言不那么强大，则应将其删除。 Common Lisp 的稳定标准不是放弃语言的结果，而是最近关于如何使核心语言比现在更好的想法很少。 并不是说它会一直如此，但 Common Lisp 目前代表了我们对编程语言的知识的顶点。 也就是说，Niche Blubs 当然有自己的位置。 有时，他们甚至为 lisp 贡献了宝贵的想法。 但是，为什么不给自己一个机会，将 Niche Blubs 实现为 Common Lisp 上的特定领域语言呢？