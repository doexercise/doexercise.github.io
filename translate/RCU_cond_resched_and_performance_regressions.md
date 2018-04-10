# RCU, cond_resched(), and performance regressions  

**By Jonathan Corbet June 24, 2014**  

Performance regressions are a constant problem for kernel developers. A seemingly innocent change might cause a significant performance degradation, but only for users and workloads that the original developer has no access to. Sometimes these regressions can lurk for years until the affected users update their kernels and notice that things are running more slowly. The good news is that the development community is responding with more testing aimed at detecting performance regressions. This testing found a classic example of this kind of bug in 3.16; the bug merits a look as an example of how hard it can be to keep things working optimally for a wide range of users.

> 성능 저하(regression)는 커널 개발자에게 지속적인 고민거리입니다. 겉으로 보기에는 아무런 문제 없는 변경이 성능저하를 유발할 수도 있지만, 이것은 원래의 개발자가 고려하지 않았던 사용자 및 작업에 대해서만 그렇습니다. 때로는 이 변경에 영향을 받는 사용자가 커널을 업데이트하고 나서 성능이 느려졌음을 인지하기 전까지 이러한 성능저하는 수 년간 숨겨져 있을 수도 있습니다. 다행히도 개발 커뮤니티는 성능 저하를 감지하기 위해 많은 테스트하고 있습니다. 이번 테스트는 3.16 버전에서 이런 종류의 버그에 대한 전형적인 예를 찾았습니다. 이 버그는 다양한 사용자들에 대하여 최적의 상태로 동작하도록 유지하는 것이 얼마나 힘든지를 보여주는 예가 될 수 있습니다.


## The birth of a regression
The kernel's read-copy-update (RCU) mechanism enables a great deal of kernel scalability by facilitating lock-free changes to data structures and batching of cleanup operations. A fundamental aspect of RCU's operation is the detection of "quiescent states" on each processor; a quiescent state is one in which no kernel code can hold a reference to any RCU-protected data structure. Initially, quiescent states were defined as times when the processor was running in user space, but things have gotten rather more complex since then. (See LWN's lengthy list of RCU articles for lots of details on how this all works).  

> 커널의 RCU (read-copy-update) 메커니즘은 데이터 구조에 대한 잠금 없는 변경을 용이하게 하고 정리 작업을 일괄 처리함으로써 많은 커널 확장성을 가능하게 합니다. RCU 동작의 기본은 각 프로세서에서 "정지 상태"를 감지하는 것입니다. "정지 상태"는 커널 코드가 RCU로 보호 된 데이터 구조를 참조할 수없는 상태입니다. 처음에 "정지 상태"는 프로세서가 사용자 공간에서 실행되는 시간으로 정의되었지만 그 이후로 상황은 다소 복잡해졌습니다. (이 모든 것이 어떻게 작동하는지에 대한 자세한 내용은 LWN의 긴 RCU 기사 목록을 참조하십시오).


The kernel's full tickless mode, which is only now becoming ready for serious use, can make the detection of quiescent states more difficult. A CPU running in the tickless mode will, due to the constraints of that mode, be running a single process. If that process stays within the kernel for a long time, no quiescent states will be observed. That, in turn, prevents RCU from declaring the end of a "grace period" and running the (possibly lengthy) set of accumulated RCU callbacks. Delayed grace periods can result in excessive latencies elsewhere in the kernel or, if things go really badly, out-of-memory problems.

> 커널의 완전한 틱리스 모드는, 이제와서야 제대로 사용할 만큼의 준비가 되어 있지만, 정지 상태를 감지하기 더 어렵게 할 수 있습니다. 틱리스 모드에서 실행중인 CPU는 해당 모드의 제약으로 인해 단일 프로세스를 실행합니다. 그 프로세스가 오랜 시간 동안 커널 내에 남아 있다면 정지 상태는 관찰되지 않습니다. 이는 차례로 RCU가 "유예 기간"의 끝을 선언하고 누적 된 RCU 콜백 세트 (길게 나타날 수 있음)를 실행하는 것을 방지합니다. 지연된 유예 기간은 커널의 다른 곳에서 과도한 대기 시간을 초래할 수 있으며, 상황이 심각하게 악화되면 메모리 부족 문제가 발생할 수 있습니다.


One might argue (as some developers did) that code that loops in the kernel in this way already has serious problems. But such situations do come about. Eric Dumazet mentioned one: a process calling exit() when it has thousands of sockets open. Each of those open sockets will result in structures being freed via RCU; that can lead to a long list of work to be done while that same process is still closing sockets and, thus, preventing RCU processing by looping in the kernel.

> 이런 방식으로 커널에서 루프하는 코드는 이미 심각한 문제가 있다고 주장 할 수도 있습니다 (일부 개발자들도 그렇듯이). 그러나 그러한 상황이 발생합니다. Eric Dumazet은 수천 개의 소켓이 열려있을 때 exit()를 호출하는 프로세스를 언급했습니다. 각각의 열린 소켓은 구조가 RCU를 통해 해제되도록합니다. 이는 동일한 프로세스가 여전히 소켓을 닫고 커널에서 루핑하여 RCU 처리를 방해하는 동안 수행해야 할 긴 작업 목록을 가져올 수 있습니다.


RCU developer Paul McKenney put together a solution to this problem based on a simple insight: the kernel already has a mechanism for allowing other things to happen while some sort of lengthy operation is in progress. Code that is known to be prone to long loops will, on occasion, call cond_resched() to give the scheduler a chance to run a higher-priority process. In the tickless situation, there will be no higher-priority process, though, so, in current kernels, cond_resched() does nothing of any use in the tickless mode.

But kernel code can only call cond_resched() in places where it can handle being scheduled out of the CPU. So it cannot be running in an atomic context and, thus, cannot hold references to any RCU-protected data structures. In other words, a call to cond_resched() marks a quiescent state; all that is needed is to tell RCU about it.

As it happens, cond_resched() is called in a lot of performance-sensitive places, so it is not possible to add a lot of overhead there. So Paul did not call into RCU to signal a quiescent state with every cond_resched() call; instead, that function was modified to increment a per-CPU counter and, using that counter, only call into RCU once for every 256 (by default) cond_resched() calls. That appeared to fix the problem with minimal overhead, so the patch was merged during the 3.16 merge window.

Soon thereafter, Dave Hansen reported that one of his benchmarks (a program which opens and closes a lot of files while doing little else) had slowed down, and that, with bisection, he had identified the cond_resched() change as the culprit. Interestingly, the problem is not with cond_resched() itself, which remained fast as intended. Instead, the change caused RCU grace periods to happen more often than before; that caused RCU callbacks to be processed in smaller batches and led to increased contention in the slab memory allocator. By changing the threshold for quiescent states from every 256 cond_resched() calls to a much larger number, Dave was able to get back to a 3.15 level of performance.

## Fixing the problem
One might argue that the proper fix is simply to raise that threshold for all users. But doing so doesn't just restore performance; it also restores the problem that the cond_resched() change was intended to fix. The challenge, then, is finding a way to fix one workload's problem without penalizing other workloads.

There is an additional challenge in that some developers would like to make cond_resched() into a complete no-op on fully preemptable kernels. After all, if the kernel is preemptable, there should be no need to poll for conditions that would require calling into the scheduler; preemption will simply take care of that when the need arises. So fixes that depend on cond_resched() continuing to do something may fail on preemptable kernels in the future.

Paul's first fix took the form of a series of patches making changes in a few places. There was still a check in cond_resched(), but that check took a different form. The RCU core was modified to take note when a specific processor holds up the conclusion of a grace period for an excessive period of time; when that condition was detected, a per-CPU flag would be set. Then, cond_resched() need only check that flag and, if it is set, note the passing of a quiescent period. That change reduced the frequency of grace periods, restoring much of the lost performance.

In addition, Paul introduced a new function called cond_resched_rcu_qs(), otherwise known as "the slow version of cond_resched()". By default, it does the same thing as ordinary cond_resched(), but the intent is that it would continue to perform the RCU grace period check even if cond_resched() is changed to skip that check — or to do nothing at all. The patch changed cond_resched() calls to cond_resched_rcu_qs() in a handful of strategic places where problems have been observed in the past.

This solution worked, but it left some developers unhappy. For those who are trying to get the most performance out of their CPUs, any overhead in a function like cond_resched() is too much. So Paul came up with a different approach that requires no checks in cond_resched() at all. Instead, when the RCU core notices that a CPU has held up the grace period for too long, it sends an inter-processor interrupt (IPI) to that processor. That IPI will be delivered when the target processor is not running in atomic context; it is, thus, another good time to note a quiescent state.

This solution might be surprising at first glance: IPIs are expensive and, thus, are not normally seen as the way to improve scalability. But this approach has two advantages: it removes the monitoring overhead from the performance-sensitive CPUs, and the IPIs only happen when a problem has been detected. So, most of the time, it should have no impact on CPUs running in the tickless mode at all. It would thus appear that this solution is preferable, and that this particular performance regression has been solved.

### How good is good enough?
At least, it would appear that way if it weren't for the fact that Dave still observes a slowdown, though it is much smaller than it was before. The solution is, thus, not perfect, but Paul is inclined to declare victory on this one anyway:

> So, given that short grace periods help other workloads (I have the scars to prove it), and given that the patch fixes some real problems, and given that the large number for rcutree.jiffies_till_sched_qs got us within 3%, shouldn't we consider this issue closed?

Dave still isn't entirely happy with the situation; he noted that the regression is closer to 10% with the default settings, and said "This change of existing behavior removes some of the benefits that my system gets out of RCU." Paul responded that he is "not at all interested in that micro-benchmark becoming the kernel's straightjacket" and sent in a pull request including the second version of the fix. If there are any real-world workloads that are adversely affected by this change, he suggested, there are a number of ways to tune the system to mitigate the problem.

Regardless of whether this issue is truly closed or not, this regression demonstrates some of the hazards of kernel development on contemporary systems. Scalability pressures lead to complex code trying to ensure that everything happens at the right time with minimal overhead. But it will never be possible for a developer to test with all possible workloads, so there will often be one that shows a surprising performance regression in response to a change. Fixing one workload may well penalize another; making changes that do not hurt any workloads may be close to impossible. But, given enough testing and attention to the problems revealed by the tests, most problems can hopefully be found and corrected before they affect production users.


* ***Original text***  <https://lwn.net/Articles/603252/>
* ***Translation Date*** 2018/04/10
* Google Translator helped me.