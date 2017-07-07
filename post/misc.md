<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
기타 정리할 내용들

<br />

# ***printk() log level***
### 로그레벨 종류 (kern_level.h)
```
KERN_EMERG(0), KERN_ALERT(1), KERN_CRIT(2), KERN_ERR(3),
KERN_WARNING(4), KERN_NOTICE(5), KERN_INFO(6), KERN_DEBUG(7)
```

### 설정 확인
```Shell
root@localhost:~# cat /proc/sys/kernel/printk
4       4       1       7
```
* 첫번째 숫자 : 현재 로그레벨
* 두번째 숫자 : printk 사용시 로그레벨을 지정하지 않은 경우 적용되는 레벨
* 세번째 숫자 : 부여할 수 있는 최소레벨. 위의 경우 "0"을 지정할 수 없음
* 네번째 숫자 : 부팅시 출력될 로그레벨

### 설정 변경
```Shell
root@localhost:~# echo 7 4 1 7 > /proc/sys/kernel/printk
```

<br>

출처 : <http://ok2513.tistory.com/9>, <http://blog.naver.com/PostView.nhn?blogId=agnazz&logNo=100122290010&parentCategoryNo=&categoryNo=40&viewDate=&isShowPopularPosts=false&from=postView>

<br />

## [**Table of Contents**](../README.md)