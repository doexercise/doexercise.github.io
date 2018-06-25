# RCU, cond_resched(), and performance regressions  

***By Jonathan Corbet June 24, 2014***

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

> RCU 개발자 Paul McKenney는 간단한 통찰력을 바탕으로 이 문제에 대한 해결책을 모았습니다. 커널은 이미 긴 작업이 진행되는 동안 다른 일이 일어나도록 허용하는 메커니즘을 이미 가지고 있습니다. 긴 루프가 발생하는 것으로 알려진 코드는 경우에 따라 cond_resched ()를 호출하여 스케줄러에게 우선 순위가 높은 프로세스를 실행할 수있는 기회를 제공합니다. 틱 없는 커널에서는 더 높은 우선 순위의 프로세스가 없으므로, 그래서 현재 커널에서는, cond_resched ()는 틱 없는 커널에서는 아무 것도 하지 않습니다.

But kernel code can only call cond_resched() in places where it can handle being scheduled out of the CPU. So it cannot be running in an atomic context and, thus, cannot hold references to any RCU-protected data structures. In other words, a call to cond_resched() marks a quiescent state; all that is needed is to tell RCU about it.

> 그러나 커널 코드는 CPU 외부에서 스케줄되는 것을 처리 할 수있는 곳에서만 cond_resched ()를 호출 할 수 있습니다. 그래서 그것은 원자적 작업에서 실행될 수 없으며 따라서 RCU로 보호 된 데이터 구조에 대한 참조를 가질 수 없습니다. 즉, cond_resched ()를 호출하면 정지 상태가 표시됩니다. 필요한 것은 RCU에게 그 사실을 알리는 것뿐입니다.

As it happens, cond_resched() is called in a lot of performance-sensitive places, so it is not possible to add a lot of overhead there. So Paul did not call into RCU to signal a quiescent state with every cond_resched() call; instead, that function was modified to increment a per-CPU counter and, using that counter, only call into RCU once for every 256 (by default) cond_resched() calls. That appeared to fix the problem with minimal overhead, so the patch was merged during the 3.16 merge window.

> 공교롭게도, cond_resched()는 성능에 민감한 많은 곳에서 호출되기 때문에 많은 오버 헤드를 추가 할 수 없습니다. 그래서 Paul은 모든 cond_resched () 호출과 함께 정지 상태를 알리기 위해 RCU를 호출하지 않았습니다. 그 대신에 그 함수는 CPU 당 카운터를 증가시키기 위해 수정되었고, 그 카운터를 사용하여 매번 256 회의 (기본값으로) cond_resched () 호출에 대해서만 RCU를 호출합니다. 이는 최소한의 오버 헤드로 문제를 해결 한 것으로 보였으며, 패치는 3.16 병합 창에서 병합되었습니다.

Soon thereafter, Dave Hansen reported that one of his benchmarks (a program which opens and closes a lot of files while doing little else) had slowed down, and that, with bisection, he had identified the cond_resched() change as the culprit. Interestingly, the problem is not with cond_resched() itself, which remained fast as intended. Instead, the change caused RCU grace periods to happen more often than before; that caused RCU callbacks to be processed in smaller batches and led to increased contention in the slab memory allocator. By changing the threshold for quiescent states from every 256 cond_resched() calls to a much larger number, Dave was able to get back to a 3.15 level of performance.

> 그 후 곧 Dave Hansen은 그의 벤치 마크 중 하나(많은 파일을 열고 닫는 프로그램)가 느려졌으며, 이분법을 통해 cond_resched ()의 변경이 범인이었음을 확인했다고 보고했습니다. 흥미롭게도 이 문제는 cond_resched() 자체가 아니며 의도 한대로 빠르게 유지됩니다. 대신, 그 변경 사항은 RCU 유예 기간을 이전보다 더 자주 발생시켰습니다. RCU 콜백이 더 작은 배치로 처리되고 슬랩 메모리 할당 자의 경쟁이 증가했습니다. 정지 상태에 대한 임계 값을 256 개 이상의 cond_resched () 호출에서 훨씬 더 많은 수로 변경함으로써 Dave는 3.15 수준의 성능으로 돌아갈 수 있었습니다.

## Fixing the problem

One might argue that the proper fix is simply to raise that threshold for all users. But doing so doesn't just restore performance; it also restores the problem that the cond_resched() change was intended to fix. The challenge, then, is finding a way to fix one workload's problem without penalizing other workloads.

> 적절한 수정은 단순히 모든 사용자에 대해 해당 임계값을 높이는 것이라고 주장할 수 있습니다. 그러나 이렇게 하면 단순히 성능을 복원하는 것이 아닙니다. 이것은 cond_resched() 변경이 고치려고 했던 문제점 또한 되돌립니다. 문제는 다른 작업 부하에 불이익을 주지 않으면 서 한 작업 부하의 문제를 해결할 수 있는 방법을 찾는 것입니다.

There is an additional challenge in that some developers would like to make cond_resched() into a complete no-op on fully preemptable kernels. After all, if the kernel is preemptable, there should be no need to poll for conditions that would require calling into the scheduler; preemption will simply take care of that when the need arises. So fixes that depend on cond_resched() continuing to do something may fail on preemptable kernels in the future.

> 일부 개발자는 완전히 선점 가능한 커널에서 cond_resched ()를 완전한 no-op로 만들고 싶어한다는 점에서 추가적인 도전 과제가 있습니다. 결국, 커널이 선점 될 수 있다면, 스케줄러에 호출해야 할 상황에서 폴링 할 필요가 없습니다. 선점은 단지 필요할 때만 처리합니다. 그러므로 cond_resched ()에 의존하는 수정은 미래에 선점 가능한 커널에서 실패 할 수도 있습니다.

Paul's first fix took the form of a series of patches making changes in a few places. There was still a check in cond_resched(), but that check took a different form. The RCU core was modified to take note when a specific processor holds up the conclusion of a grace period for an excessive period of time; when that condition was detected, a per-CPU flag would be set. Then, cond_resched() need only check that flag and, if it is set, note the passing of a quiescent period. That change reduced the frequency of grace periods, restoring much of the lost performance.

> Paul의 첫 번째 수정은 일련의 패치를 몇 군데에서 변경하는 형식을 취했습니다. cond_resched ()에서 여전히 검사가 있었지만 검사는 다른 형식을 취했습니다. RCU 코어는 특정 프로세서가 주어진 시간을 초과하도록 점유할 때 이를 확인하도록 수정되었습니다. 해당 조건이 감지되면 CPU 당 플래그가 설정됩니다. 그런 다음 cond_resched ()는 해당 플래그를 확인하기만 하면 됩니다. 플래그가 설정되어 있으면 정지 기간이 경과 한 것을 확인합니다. 이러한 변경으로 인해 유예 기간의 빈도가 줄어들어 손실 된 성능이 상당 부분 복원되었습니다.

In addition, Paul introduced a new function called cond_resched_rcu_qs(), otherwise known as "the slow version of cond_resched()". By default, it does the same thing as ordinary cond_resched(), but the intent is that it would continue to perform the RCU grace period check even if cond_resched() is changed to skip that check — or to do nothing at all. The patch changed cond_resched() calls to cond_resched_rcu_qs() in a handful of strategic places where problems have been observed in the past.

> 또한 Paul은 cond_resched_rcu_qs ()라는 새로운 함수를 도입했습니다.이 함수는 "느린 버전의 cond_resched ()"로도 알려져 있습니다. 이것은 기본적으로 일반적인 cond_resched ()와 동일한 작업을 수행하지만 cond_resched ()가 해당 검사를 건너 뛰거나 전혀 수행하지 않을 때에도 RCU 유예 기간 검사를 계속 수행 할 것입니다. 패치는 과거에 문제가 관찰 된 소수의 전략적 장소에서 cond_resched ()가 cond_resched_rcu_qs ()을 호출하도록 변경했습니다.

This solution worked, but it left some developers unhappy. For those who are trying to get the most performance out of their CPUs, any overhead in a function like cond_resched() is too much. So Paul came up with a different approach that requires no checks in cond_resched() at all. Instead, when the RCU core notices that a CPU has held up the grace period for too long, it sends an inter-processor interrupt (IPI) to that processor. That IPI will be delivered when the target processor is not running in atomic context; it is, thus, another good time to note a quiescent state.

> 이 솔루션은 효과가 있었지만 일부 개발자는 불만을 나타냈습니다. CPU에서 최대 성능을 얻으려는 사람들은 cond_resched ()와 같은 함수에서 오버 헤드가 너무 많습니다. 그래서 Paul은 cond_resched ()에서 전혀 검사할 필요가 없는 다른 접근법을 제안했습니다. 대신 RCU 코어가 CPU가 너무 오랫동안 유예 기간을 점유했다는 것을 알게 되면 CPU에 프로세서 간 인터럽트 (IPI)를 보냅니다. 타겟 프로세서가 원자적 컨텍스트에서 실행되지 않을 때 그 IPI가 전달됩니다. 그러므로, 정지 상태를 확인하는 또 다른 좋은 시기입니다.

This solution might be surprising at first glance: IPIs are expensive and, thus, are not normally seen as the way to improve scalability. But this approach has two advantages: it removes the monitoring overhead from the performance-sensitive CPUs, and the IPIs only happen when a problem has been detected. So, most of the time, it should have no impact on CPUs running in the tickless mode at all. It would thus appear that this solution is preferable, and that this particular performance regression has been solved.

> 이 솔루션은 언뜻보기에 놀라운 것일 수 있습니다. IPI는 비용이 많이 들고 일반적으로 확장 성을 개선하는 방법으로 볼 수 없습니다. 그러나 이 접근법에는 두가지 이점이 있습니다. 성능에 민감한 CPU에서 모니터링 오버 헤드가 제거된다는 점과 IPI는 문제가 감지 된 경우에만 발생한다는 점입니다. 따라서 대부분의 경우 틱없는 모드로 실행되는 CPU에는 아무런 영향을 미치지 않습니다. 따라서이 솔루션이 바람직하며 이 특정 성능 저하가 해결 된 것으로 보입니다.

### How good is good enough?

At least, it would appear that way if it weren't for the fact that Dave still observes a slowdown, though it is much smaller than it was before. The solution is, thus, not perfect, but Paul is inclined to declare victory on this one anyway:

> 데이브가 성능저하가 이전보다 훨씬 적어졌다고 하더라도 이 점을 여전히 지켜보고 있다는 사실만 제외하면, 그것은 잘 되었다고 느껴질 것입니다. 해결책은 완벽한 것이 아니지만 폴은 어쨌든 이 일에 승리를 선포합니다.

\> So, given that short grace periods help other workloads (I have the scars to prove it), and given that the patch fixes some real problems, and given that the large number for rcutree.jiffies_till_sched_qs got us within 3%, shouldn't we consider this issue closed?

> 그래서 짧은 유예 기간이 다른 작업에 유용하고(나는 그것을 증명할 증거가 있음), 패치가 몇가지 실제 문제를 해결하고, rcutree.jiffies_till_sched_qs의 수가 많으면 3 % 이내에 있게 되므로, 이 문제가 해결 된 것으로 생각하십니까?

Dave still isn't entirely happy with the situation; he noted that the regression is closer to 10% with the default settings, and said "This change of existing behavior removes some of the benefits that my system gets out of RCU." Paul responded that he is "not at all interested in that micro-benchmark becoming the kernel's straightjacket" and sent in a pull request including the second version of the fix. If there are any real-world workloads that are adversely affected by this change, he suggested, there are a number of ways to tune the system to mitigate the problem.

> Dave는 여전히 상황에 완전히 만족하지 않습니다. 그는 기본 설정값으로 된 상태에서  회기가 10 %에 가까워 졌다고 말하면서 "이 기존 행동의 변화로 인해 시스템이 RCU에서 얻는 이점 중 일부가 제거됩니다." 라고 했습니다. Paul은 "마이크로 벤치 마크가 커널의 직통 전화가 되는 것에 전혀 관심이 없다"고 응답하고 수정본의 두 번째 버전을 포함하여 요청을 보냈습니다. 이 변경으로 인해 영향을 받는 실제 작업이 있다면, 문제점을 완화하기 위한 여러가지 시스템 튜닝 방법이 있다고 그는 제안했습니다.

Regardless of whether this issue is truly closed or not, this regression demonstrates some of the hazards of kernel development on contemporary systems. Scalability pressures lead to complex code trying to ensure that everything happens at the right time with minimal overhead. But it will never be possible for a developer to test with all possible workloads, so there will often be one that shows a surprising performance regression in response to a change. Fixing one workload may well penalize another; making changes that do not hurt any workloads may be close to impossible. But, given enough testing and attention to the problems revealed by the tests, most problems can hopefully be found and corrected before they affect production users.

> 이 문제가 진정으로 종결되었는지 여부에 관계없이 이 회귀는 현대 시스템에서 커널 개발의 위험을 보여줍니다. 확장성에 대한 압박은 최소한의 오버 헤드로 모든 것이 적절한 시간에 발생하도록 보장하기 위해 복잡한 코드로 이어집니다. 그러나 개발자가 가능한 모든 작업 부하로 테스트 할 수는 없으므로 변경 사항에 대한 응답으로 놀라운 성능 회귀를 나타내는 경우가 종종 있습니다. 한 워크로드를 수정하면 다른 워크로드를 불이익을 받을 수 있습니다. 어떤 작업 부하에도 해를 끼치 지 않는 변경은 거의 불가능할 수 있습니다. 그러나 테스트에서 밝혀진 문제에 대한 충분한 테스트와 주의를 기울이면 대부분의 문제는 프로덕션 사용자에게 영향을 미치기 전에 찾아서 수정할 수 있습니다.

* ***Original text***  <https://lwn.net/Articles/603252/>
* ***Translation Date*** 2018/04/10
* Google Translator helped me.