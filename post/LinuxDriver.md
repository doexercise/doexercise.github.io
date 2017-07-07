<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
Linux Device Driver 제작에 대한 노트
* [tasklet](#tasklet)
* [workqueue](#workqueue)
* [completion](#completion)  
<br />

# ***Bottom half***
Interrupt 처리를 전반부 처리(Top half)로 봤을 때 지연된 처리를 후반부 처리(Bottom half)라 하며, kernel 2.6 이후 이에 대한 구현은 softirq, [tasklet](#tasklet), [workqueue](#workqueue) 를 사용한다.

## tasklet
tasklet은 softirq 로 구현되어있다. 그러나 거의 softirq 보다 tasklet으로 처리한다. softirq 와 다르게 tasklet 은 실행의 직렬화를 보장한다. 즉 동일 tasklet 에서는 동기화가 필요하지 않다. <br> 
workqueue에 비해 높은 우선순위를 가지며 휴면 상태로 전환될 일이 없으면 workqueue보다 tasklet으로 처리한다.<br>  
그리고 이것은 kernel 2.6 이전의 taskqueue와 관계가 없다. 

### Basic example
```C
#include <linux/module.h>
#include <linux/interrupt.h>

void tasklet_handler_fn(unsigned long data);
int __init ckun_init(void);
void __exit ckun_exit(void);

int param;

/**
 * DECLARE_TASKLET(name, func, data);
 *      - name : 생성할 tasklet_struct 구조체의 이름.
 *      - func : tasklet의 핸들러 함수
 *      - data : tasklet handler에 전달될 인자의 주소. 즉 인자의 포인터.
 */
DECLARE_TASKLET(my_tasklet_s, tasklet_handler_fn, (unsigned long)&param);

void tasklet_handler_fn(unsigned long data)
{
        int * val = (int *)data;

        printk("data : %d\n", *val);    // 1
        printk("addr : %p\n", val);     // int param 의 주소와 같음

        (*val)++;

        return;
}

int __init ckun_init(void)
{
        printk("data : %d\n", param);   // 0
        printk("addr : %p\n", &param);

        param++;

        /**
         * 커널에 스케줄링을 요청. 동일한 tasklet은 동시에 중복해서 실행되지 않음.
         * 이미 실행중인 경우 재차 스케줄링을 요청하게 됨.
         */
        tasklet_schedule(&my_tasklet_s);

        return 0;
}

void __exit ckun_exit(void)
{
        printk("data : %d\n", )         // 2

        // tasklet 삭제
        tasklet_kill(&my_tasklet_s);

        return;
}

module_init(ckun_init);
module_exit(ckun_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ckun");
```

<br />

## workqueue  
일반적인 커널 스레드 형태로 동작하며, 기본 workqueue 를 사용할 수도 있고 직접 workqueue 를 생성할 수도 있다.  
기본 workqueue 는 kevents/n(또는 kworker) 프로세스로 동작하며, 직접 생성하는 workqueue 도 일반 커널 스레드와 크게 다르지 않다. <U>다만 workqueue 는 프로세서 별로 스레드가 생성된다.</U>  
workqueue 대신 일반 커널 스레드를 생성해도 작업에 제약은 없다.

### 기본 workqueue 사용 예시
```C
#include <linux/module.h>

void my_workqueue_fn(struct work_struct * my_wq);
int __init ckun_init(void);
void __exit ckun_exit(void);

/**
 * my_work_s 라는 work_struct 구조체를 생성
 * workqueue 가 스케줄링 되면 my_workqueue_fn 핸들러 실행
 */
DECLARE_WORK(my_work_s, my_workqueue_fn);

void my_workqueue_fn(struct work_struct * my_wq)
{
        printk("workqueue is called\n");
 
        return;
}

int __init ckun_init(void)
{
        // workqueue 스케줄링 등록
        schedule_work(&my_work_s);

        return 0;
}

void __exit ckun_exit(void)
{
        // 스케줄링 된 모든 작업이 종료될 때까지 반환하지 않음
        // 작업 취소는 schedule_delayed_work() 사용
        flush_scheduled_work();

        return;
}

module_init(ckun_init);
module_exit(ckun_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ckun");
```

### 기본 workqueue에서 데이터 전달
```C
#include <linux/module.h>


void my_workqueue_fn(struct work_struct * ptr);
int __init ckun_init(void);
void __exit ckun_exit(void);

/**
 * work_struct 와 전달하고자 하는 데이터를 포함하는 구조체 선언.
 * workqueue 초기화는 work_struct 로 하고, 
 * 'container_of' 를 통해 data에 접근한다.
 */
struct my_work_t {
        struct work_struct      m_work;
        int                     m_data;
};

// 전역변수로 선언하여 굳이 container_of 가 필요 없을 수도 있지만
// 동적할당(malloc)을 받은 경우 container_of 가 유용함
struct my_work_t my_work_s;

void my_workqueue_fn(struct work_struct * ptr)
{
        // container_of(포인터, 구조체 타입, 포인터의 멤버변수 이름)
        struct my_work_t * p = container_of(ptr, struct my_work_t, m_work);

        printk("data : %d\n", p->m_data);       // 10

        return;
}

int __init ckun_init(void)
{
        // DECLARE_WORK 처럼 함수 밖에서 처리하면 에러
        INIT_WORK(&my_work_s.m_work, my_workqueue_fn);

        my_work_s.m_data = 10;

        schedule_work(&my_work_s.m_work);

        return 0;
}

void __exit ckun_exit(void)
{
        flush_scheduled_work();

        return;
}

module_init(ckun_init);
module_exit(ckun_exit);  
```

## 새로운 workqueue 생성
생성 `create_workqueue`, 소멸 `destroy_workqueue`, 스케줄링 `queue_work` 의 함수명만 다를뿐 기본 workqueue와 동일하다고 보면 된다.
```C
#include <linux/module.h>

void my_workqueue_fn(struct work_struct * ptr);
int __init ckun_init(void);
void __exit ckun_exit(void);

struct workqueue_struct * my_wq;
DECLARE_WORK(my_work, my_workqueue_fn);

void my_workqueue_fn(struct work_struct * ptr)
{
        printk("workqueue is called\n");

        return;
}

int __init ckun_init(void)
{
        // 프로세서 마다 하나의 workqueue 가 생성된다.
        // ps 명령어로 보면 이 이름의 프로세스가 보인다.
        my_wq = create_workqueue("ckun_work");

        queue_work(my_wq, &my_work);

        return 0;
}

void __exit ckun_exit(void)
{
        // 스케줄링 된 작업을 처리
        flush_workqueue(my_wq);

        // 생성된 workqueue 를 제거
        destroy_workqueue(my_wq);

        return;
}

module_init(ckun_init);
module_exit(ckun_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ckun");
```

다만 프로세서 수와 관계없이 오직 한개의 작업 스레드를 생성하고자 하면 아래 함수를 사용하라.
```C
struct workqueue_struct * my_wq;
my_wq = create_singlethread_workqueue("ckun_work");
```

<br>  


## completion
일종의 이벤트

### Basic Example
두개의 workqueue를 생성해서 첫번째 workqueue는 completion 대기하고
두번째 workqueue는 완료이벤트 발생하는 예제이다.
```C
#include <linux/module.h>

void my_workqueue_fn(struct work_struct * ptr);
int __init ckun_init(void);
void __exit ckun_exit(void);

struct workqueue_struct *my_wq1, *my_wq2;

DECLARE_WORK(my_work1, my_workqueue_fn);
DECLARE_WORK(my_work2, my_workqueue_fn);

/**
 * 정적 생성은 DECLARE_COMPLETION(name)
 * 동적 생성은 init_completion(struct completion *)
 */
DECLARE_COMPLETION(my_comp);

void my_workqueue_fn(struct work_struct * ptr)
{
        if (ptr == &my_work1) {
                wait_for_completion(&my_comp);
        }
        else if (ptr == &my_work2) {
                complete(&my_comp);
        }

        return;
}

int __init ckun_init(void)
{
        my_wq1 = create_singlethread_workqueue("ckun_work1");
        my_wq2 = create_singlethread_workqueue("ckun_work2");

        queue_work(my_wq1, &my_work1);
        queue_work(my_wq2, &my_work2);

        return 0;
}

void __exit ckun_exit(void)
{
        flush_workqueue(my_wq1);
        flush_workqueue(my_wq2);

        destroy_workqueue(my_wq1);
        destroy_workqueue(my_wq2);

        return;
}

module_init(ckun_init);
module_exit(ckun_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ckun");
```

<br>   

## [**Table of Contents**](../README.md)