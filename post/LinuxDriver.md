<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
Linux Device Driver 제작에 대한 노트

<br />

# ***Bottom half***
Interrupt 처리를 전반부 처리(Top half)로 봤을 때 지연된 처리를 후반부 처리(Bottom half)라 하며, kernel 2.6 이후 이에 대한 구현은 softirq, [tasklet](#tasklet), workqueue 를 사용한다.

## tasklet
tasklet은 softirq 로 구현되어있다. 이것은 kernel 2.6 이전의 taskqueue와 관계가 없다.

### 정적 생성
```C
/**
 * name : 생성할 tasklet_struct 구조체의 이름.
 * func : tasklet의 핸들러 함수
 * data : tasklet handler에 전달될 인자의 주소. 즉 인자의 포인터.
 */
DECLARE_TASKLET(name, func, data);
```

### tasklet 실행 요청(스케줄 등록)
```C
/**
 * 커널에 스케줄링을 요청. 동일한 tasklet은 동시에 중복해서 실행되지 않음.
 * 이미 실행중인 경우 재차 스케줄링을 요청하게 됨.
 */
tasklet_schedule(struct tasklet_struct * t);
```

### tasklet 삭제
```C
tasklet_kill(struct tasklet_struct * t);
```

### Basic example
```C
#include <linux/module.h>
#include <linux/interrupt.h>

void tasklet_handler_fn(unsigned long data);
int __init ckun_init(void);
void __exit ckun_exit(void);

int param;

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

        tasklet_schedule(&my_tasklet_s);

        return 0;
}

void __exit ckun_exit(void)
{
        printk("data : %d\n", )         // 2

        tasklet_kill(&my_tasklet_s);

        return;
}

module_init(ckun_init);
module_exit(ckun_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ckun");
```

<br />

## [**Table of Contents**](../README.md)