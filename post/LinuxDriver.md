<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
Linux Device Driver 제작에 대한 노트  
* [**bottom half**](#bottom-half)      
  * [tasklet](#tasklet)     
  * [workqueue](#workqueue)    
* [**completion**](#completion)  
* [**debugging**](#debugging)  
* [**proc**](#proc)

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
        // schedule_delayed_work()에 대한 작업취소는 cancel_delayed_work() 사용
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


# ***completion***
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

# ***debugging***
리눅스에서 커널 모듈 디버깅은 꽤 불편한 듯

## Example 1  
> 에러 메세지

```
[  120.908362] BUG: unable to handle kernel paging request at ffff881f339af408
[  120.908412] IP: [<ffffffffc03a964b>] cleanup_module+0x17/0x9cc [ckun]
<생략>
[  120.909793] Call Trace:
[  120.909814]  [<ffffffff81108705>] SyS_delete_module+0x1b5/0x210
[  120.909849]  [<ffffffff81840b72>] entry_SYSCALL_64_fastpath+0x16/0x71
```

> `cleanup_module` 이게 뭐지? 난 이런 함수를 쓰지 않았는데?  

kernel 2.4 이하에서는 `module_init`, `module_exit` 대신 `init_module`, `cleanup_module` 을 사용.   
내 경우 `module_exit(ck_exit);` ->  `void __exit ck_exit(void);` 이 함수의 심볼이 `cleanup_module` 이다.

<br>

> `cleanup_module+0x17/0x9cc` : cleanup_module 함수로부터 오프셋 0x17 지점에서 문제 발생. 0x17 지점은 아래그림에서 <+23> 이다.(16진수 -> 10진수)

```
root@localhost:~# gdb ckun.ko
(gdb) disassemble cleanup_module
Dump of assembler code for function cleanup_module:
   0x00000000000006cf <+0>:     push   %rbp
   0x00000000000006d0 <+1>:     mov    %rsp,%rbp
   0x00000000000006d3 <+4>:     push   %rbx
   0x00000000000006d4 <+5>:     xor    %ebx,%ebx
   0x00000000000006d6 <+7>:     cmp    %ebx,0x0(%rip)   # 0x6dc <cleanup_module+13>
   0x00000000000006dc <+13>:    jle    0x6f9 <cleanup_module+42>
   0x00000000000006de <+15>:    mov    0x0(,%rbx,8),%rax
   0x00000000000006e6 <+23>:    mov    0x8(%rax),%rdi
   0x00000000000006ea <+27>:    test   %rdi,%rdi
   0x00000000000006ed <+30>:    je     0x6f4 <cleanup_module+37>
   0x00000000000006ef <+32>:    callq  0x6f4 <cleanup_module+37>
   0x00000000000006f4 <+37>:    inc    %rbx
   0x00000000000006f7 <+40>:    jmp    0x6d6 <cleanup_module+7>
   0x00000000000006f9 <+42>:    mov    $0x0,%rdi
   0x0000000000000700 <+49>:    callq  0x705 <cleanup_module+54>
   0x0000000000000705 <+54>:    callq  0x70a <cleanup_module+59>
   0x000000000000070a <+59>:    mov    $0x0,%rdi
   0x0000000000000711 <+66>:    callq  0x716 <cleanup_module+71>
   0x0000000000000716 <+71>:    pop    %rbx
   0x0000000000000717 <+72>:    pop    %rbp
   0x0000000000000718 <+73>:    retq
End of assembler dump.
```

<br>

> 여기서는 다른 함수에서 변수를 초기화 하면서 `p = (void *)a` 라고 해야 할 것을 `p = (void *)&a` 라고 하여 전혀 다른 위치를 참조하고 있었고, 결국 모듈 제거시 에러 발생

<br />

# ***proc***
proc 파일 시스템

## umode_t
* proc_create() 함수원형, linux/proc_fs.h

	```C
	static inline struct proc_dir_entry *proc_create(
	         const char *name, 
	         umode_t mode, 
	         struct proc_dir_entry *parent,
	         const struct file_operations *proc_fops
	);
	```  

<br>

* 보통 `proc_create("hello", 0, NULL, &proc_fops);` 사용. mode 가 0이면 permission 0444 로 설정됨. 물론 `proc_create("hello", 0444, NULL, &proc_fops);` 라고 해도 됨
  > 출처 : <https://stackoverflow.com/questions/28664971/why-is-does-proc-create-mode-argument-0-correspond-to-0444>


<br>

* 퍼미션에 대한 정의, linux/stat.h (uapi/linux/stat.h)
	```C
	 #define S_IRWXU 00700	// R + W + X => user
	 #define S_IRUSR 00400	// R only
	 #define S_IWUSR 00200	// W only
	 #define S_IXUSR 00100	// X only
	
	 #define S_IRWXG 00070	// R + W + X => group
	 #define S_IRGRP 00040
	 #define S_IWGRP 00020
	 #define S_IXGRP 00010
	
	 #define S_IRWXO 00007	// R + W + X => other
	 #define S_IROTH 00004
	 #define S_IWOTH 00002
	 #define S_IXOTH 00001
	 ```

<br>

* 파일 형식에 대한 정의, linux/stat.h (uapi/linux/stat.h)
	> S_IFREG  : 일반 파일  
	> S_IFCHR  : 문자 파일  
	> S_IFBLK  : 블럭 파일  
	> S_IFSOCK : Unix Domain Sock 파일    
	> <U>단, proc 파일은 S_IFREG 이외에는 지정할 수 없고 따라서 별도로 지정할 필요도 없다.</U>  
	> 출처 : <https://www.joinc.co.kr/w/man/2/mknod>

<br />

## proc 생성/삭제
```C
const char *name = "ckun";
umode_t mode = 0444;
struct proc_dir_entry *parent;
struct proc_dir_entry *child;
struct file_operations fops = {
	.owner	= THIS_MOUDLE,
	.open	= proc_open_fn,
	.read 	= proc_read_fn,
	.write	= proc_write_fn,
};

// 두번째 인자에 NULL을 전달하면 /proc 밑에 디렉토리를 생성한다. /proc/ckun
parent = proc_mkdir("ckun", NULL);

// 위에서 생성한 디렉토리를 부모로 하는 파일이 생성된다. /proc/ckun/ckun1
child = proc_create("ckun1", 0444, parent, &fops);
```
```C
// /proc/ckun 을 삭제한다.
remove_proc_entry("ckun", NULL);
```
```C
// /proc/ckun/ckun1 을 삭제한다.
remove_proc_entry("ckun1", parent);
```
```C
// /proc/ckun 이하 모든 것을 삭제한다.
remove_proc_subtree("ckun", NULL);
```

<br>

## proc_create() vs proc_create_data()
proc_create_data 는 void *data 인자를 하나 더 받는다. proc_create 는 마지막 인자를 NULL로 하여 proc_create_data 를 호출하는 wrapper 일 뿐이다.
```C
proc_create_data(char *name, 
		umode_t mode, 
		struct proc_dir_entry *parent, 
		struct file_operations *fops, 
		void *data);
```

<br>

## 장치별로 개별 데이터 전달하기
예를 들어, 아래와 같이 proc를 구성했다고 할때 개별 데이터를 보관하기 위한 방법을 알아보자
```
/proc ── ckun  ─┬─ dev1  
		├─ dev2  
      		└─ dev3  
```

```C
// 장치별 개별 데이터(보통은 이렇게 정적으로 생성하지 않고 probe 단계에서 kmalloc 한다)
struct my_contents_t my_contents[3];
```
```C
// 디렉토리 및 파일 생성. 참고로 read/write 모두 .open 함수가 먼저 실행된다.
struct file_operations fops = {
	.owner	= THIS_MOUDLE,
	.open	= proc_open_fn,
	.read	= proc_read_fn,
	.write	= proc_write_fn,
};
parent = proc_mkdir("ckun", NULL);
proc_create_data("dev1", 0444, parent, &fops, &my_contents[0]);
proc_create_data("dev2", 0444, parent, &fops, &my_contents[1]);
proc_create_data("dev3", 0444, parent, &fops, &my_contents[2]);
```
```C
/**
 * single_open 함수를 통해 seq_file 구조체를 생성하고 seq_fn 함수 호출.
 * PDE_DATA 매크로는 inode 로 부터 proc_create_data() 에서 지정했던
 * void *data 주소를 가져온다. 
 * 물론 proc_create()로 생성하고 이곳에서 직접 void *data 를 설정할 수도 있다. 
 * 그러나 그렇게 하려면 dev1, dev2, dev3 이 모두 다른 callback 함수를 사용해야 
 * 어떤 장치에 대한 호출인지 알 수 있을거다.
 */
int proc_open_fn(struct inode *inode, struct file *file)
{
	return single_open(file, seq_fn, PDE_DATA(inode));
}

/**
 * 앞서 언급했듯이 read/write 모두 이 함수가 호출이 된다. 
 * read에 대한 콜백함수를 사용자가 직접 작성하지 않을 수 있고, 
 * 이 경우 .read = seq_read 로 설정해야 하는데,
 * 이렇게 설정된 경우에는 seq_print() 를 통해 사용자 화면(터미널)에
 * 필요한 내용을 출력할 수 있다. 
 * 즉, read 의 경우 .open 만 직접 작성하고 .read = seq_read 로 설정하면
 * 충분히 목적한 바를 사용자에게 전달할 수 있다.
 * 반대로, read 함수를 직접 작성하여 지정한 경우 seq_printf()는 화면에 출력하지 않는다.
 * write 의 경우에는 기본 함수가 무엇인지 확인이 안된다. seq_write()는 함수원형이 다름.
 */
int seq_fn(struct seq_file *m, void *v)
{
	// 앞서 지정한 void *data는 다음과 같이 가져올 수 있다.
	struct my_contents *p = (struct my_contents *)m->private;

	// 이 예제에서는 .read 를 직접 지정하였기 때문에
	// seq_printf() 는 화면에 내용을 출력하지 않는다.
	seq_printf(m, "hi proc\n");
	seq_printf(m, "Data : %d\n", p->cnt);
	return 0;
}

/**
 * single_open() 을 사용하지 않았거나, void *data 를 지정하지 않은 경우 
 * seq_file * 를 설정하는 부분에서 oops 발생함.
 * .read 콜백은 처리한 데이터 사이즈를 리턴해야 데이터 전달이 수행되고, 
 * 0을 리턴해야 종료된다. 0 이외의 값(0보다 큰?)을 리턴하면 계속 호출된다.
 * 따라서 ret 값을 static 으로 선언하고 첫번째 호출시에는 처리한 데이터의 크기를 리턴하고
 * 다시 호출되었을때 0을 리턴함으로써 작업을 종료시킨다.
 * read 함수 참고 : https://blog.nyanpasu.me/a-proc-file-example/
 */
ssize_t proc_read_fn(struct file *file, char __user *buf, size_t size, loff_t *ppos)
{
	// 앞서 지정한 void *data 를 가져오는 방법
	struct seq_file *m = (struct seq_file *)file->private_data;
	struct my_contents *p = m->private;

	static int ret = 0;
	if (ret == 0) {
		copy_to_user(buf, p, sizeof(*p));
		ret = sizeof(*p);
	} else {
		ret = 0;
	}

	return ret;
}

/**
 * write의 경우 "echo 1 > /proc/ckun/dev1" 처럼 사용되는 경우를 가정해 보았다.
 * read 의 경우와 다르게 전달받은 데이터 크기를 리턴함으로써 함수는 종료된다.
 */
ssize_t proc_write_fn(struct file *file, const char __user *buf, size_t size, loff_t *ppos)
{
        // proc_read_fn() 에서와 달리 이렇게 해도 된다.
        // 결국 file 에서 inode 를 찾고, inode에서 proc_dir_entry,
        // 최종적으로 proc_dir_entry 의 data 멤버를 리턴하는 것.
	char *p = (char *)PDE_DATE(file_inode(file));

	// for preventing buffer overrun
	if (size > BUFFER_SIZE) 
		size = BUFFER_SIZE - 1;

	copy_from_user(p, buf, size);
	if (p[size-1] == '\n')
		p[size-1] = '\0';
	else 
		p[size] = '\0';

	printk("Data : %s\n", p);

	return size;
}
```

<br>

## [**Table of Contents**](../README.md)