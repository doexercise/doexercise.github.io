<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
기타 정리할 내용들  
* [**printk()**](#printk()-log-level)
* [**Variadic Macro**](#Variadic-Macro)
* [**How to access FPGA on Zynq**](#How-to-access-FPGA-on-Zynq)
<br />

# ***printk() log level***
### 로그레벨 (kern_level.h)
```
KERN_EMERG(0), KERN_ALERT(1), KERN_CRIT(2), KERN_ERR(3),
KERN_WARNING(4), KERN_NOTICE(5), KERN_INFO(6), KERN_DEBUG(7)
```

### 설정 확인
```
root@localhost:~# cat /proc/sys/kernel/printk
4       4       1       7
```
* 첫번째 숫자 : 현재 로그레벨
* 두번째 숫자 : printk 사용시 로그레벨을 지정하지 않은 경우 적용되는 레벨
* 세번째 숫자 : 부여할 수 있는 최소레벨. 위의 경우 "0"을 지정할 수 없음
* 네번째 숫자 : 부팅시 출력될 로그레벨

### 설정 변경
```
root@localhost:~# echo 7 4 1 7 > /proc/sys/kernel/printk
```

<br>

### 출처 : 
 * <http://ok2513.tistory.com/9> 
 * <http://blog.naver.com/PostView.nhn?blogId=agnazz&logNo=100122290010&parentCategoryNo=&categoryNo=40&viewDate=&isShowPopularPosts=false&from=postView>

<br />



# ***Variadic Macro***
### gcc
```C
#define DEBUG(fmt, ...)    \
        printk(KERN_DEBUG"[ckun] Debug " fmt ", %s, %d\n", \
        ##__VA_ARGS__, __func__, __LINE__)
```
> `##__VA_ARGS__` 가 아니라 `__VA_ARGS__` 로 나오는 자료가 있는데, C99 에서만 가능하다고 에러 뜸
> 그렇다고 `ccflags-y += -std=c99` 로 하면 다른 곳에서 에러가 생김

<br>

### VC++
```C
#define DEBUG_INFO(fmt, ...)	\
	DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL, \
        "ckun(%s, %d): " fmt, __FUNCTION__, __LINE__, __VA_ARGS__)
```

<br>

# ***How to access FPGA on Zynq***
### Sample Code
```C
/**
* Refer to following links :
*       http://svenand.blogdrives.com/files/gpio-dev-mem-test.c
*       http://www.wiki.xilinx.com/Linux+User+Mode+Pseudo+Driver
*       https://forums.xilinx.com/t5/Zynq-All-Programmable-SoC/how-to-read-write-from-to-a-register-in-Linux/td-p/581391
*/

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>

#define GPIO_BASE_ADDRESS       0x41200000
#define MAP_SIZE                0x10000UL       // 64k
#define MAP_MASK                (MAP_SIZE -1)
#define LED                     0x01

int main(int argc, char ** argv)
{
        volatile void *mapped_base, *mapped_dev_base;
        int fd  = 0;
        int val = 0;
        int i   = 0;
        int c   = 0;

        int offset = 0;
        int channel_id = 1;
        int interval = 500;
        int count = 100;


        while ((c = getopt(argc, argv, "c:i:n:h")) != -1) {
                switch(c) {
                case 'c':
                        channel_id = atoi(optarg);
                        if (channel_id != 1 && channel_id != 2) {
                                printf("Invalid channel id\n");
                                channel_id = 1;
                        }
                        break;

                case 'i':
                        interval = atoi(optarg);
                        if (interval < 10 || interval > 3000) {
                                printf("Invalid interval\n");
                                interval = 500;
                        }
                        break;

                case 'n':
                        count = atoi(optarg);
                        if (count < 1) {
                                printf("Invalid count\n");
                                count = 100;
                        }
                        break;

                case 'h':
                case '?':
                        printf("Usage :                                         \n");
                        printf("        -c      channel id (1 or 2)             \n");
                        printf("        -i      interval (10 <= i <= 3000)      \n");
                        printf("                millisecond                     \n");
                        printf("        -n      repeat count (0 < c)            \n");
                        printf("        -h      this screen                     \n");
                        printf("                                                \n");
                        printf("        default c = 1                           \n");
                        printf("                i = 500                         \n");
                        printf("                n = 100                         \n");
                        return -1;
                }
        }

        offset = (channel_id == 1) ? 0 : 8;

        fd = open("/dev/mem", O_RDWR | O_SYNC);
        if (fd < 1) {
                perror("Not found /dev/mem");
                return -1;
        }

        mapped_base = mmap(NULL, MAP_SIZE, PROT_WRITE | PROT_READ, MAP_SHARED,
                                fd, (GPIO_BASE_ADDRESS + offset) & ~MAP_MASK);
        if (mapped_base == MAP_FAILED) {
                perror("mmap");
                goto EXIT;
        }

        mapped_dev_base = mapped_base + (GPIO_BASE_ADDRESS & MAP_MASK);
        printf("dev base addr : 0x%p\n", mapped_dev_base);

        while (i < count) {
                val = *((unsigned *)(mapped_dev_base + 0));

                if (val & LED) {
                        *((unsigned *)(mapped_dev_base + 0)) = val & ~LED;

                }
                else {
                        *((unsigned *)(mapped_dev_base + 0)) = val | LED;
                }

                printf("(%5d) val       : %d\n", i++, val);
                usleep(interval * 1000);
        }

        if (munmap(mapped_base, MAP_SIZE) == MAP_FAILED) {
                perror("unmap");
                goto EXIT;
        }

EXIT:
        close(fd);
        return 0;
}
```
<br>

## [**Table of Contents**](../README.md)