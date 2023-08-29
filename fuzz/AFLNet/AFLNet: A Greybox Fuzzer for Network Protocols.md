这篇论文没有什么创新点，主要是将AFL应用到网络协议的fuzzing上。

整体架构如下图：

![](https://raw.githubusercontent.com/mark0-cn/blog_img/master/img/202308292008323.png)

维护一个状态机（State Machine Learner），根据server返回的状态值判断新路径的发现。

Sequence Mutator，并不会将整个片段进行变异。

1) the prefix M1 is required to reach the selected state s.
2) the candidate subsequence M2 contains all messages that can be executed after M1 while still remaining in s.
3) the suffix M3 is simply the left-over subsequence such that 〈M1, M2, M3〉 = M. 

The mutated message sequence M ′ = 〈M1, mutate(M2), M3〉. By maintaining the original subsequence M1, M ′ will still reach the state s which is the state that the fuzzer is currently focusing on. The mutated candidate subsequence mutate(M2) produces an alternative sequence of messages upon the choosen state s. In our initial experiments, we observed that the alternative requests may not be observable "now", but propagate to later responses. Hence, AFLNET continues with the execution of the suffix M3.