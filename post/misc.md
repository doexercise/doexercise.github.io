<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
기타 정리할 내용들

<br />

# ***printk***
#### 로그레벨은 다음과 같다. (kern_level.h)
```
KERN_EMERG(0), KERN_ALERT(1), KERN_CRIT(2), KERN_ERR(3),
KERN_WARNING(4), KERN_NOTICE(5), KERN_INFO(6), KERN_DEBUG(7)
```



#### 다음 명령을 실행하면 몇가지 사항을 확인 할 수 있다.
```Shell
cat /proc/sys/kernel/printk
4       4       1       7
```
* 첫번째 숫자 : 현재 로그레벨
* 두번째 숫자 : printk 사용시 로그레벨을 지정하지 않은 경우 적용되는 레벨
* 세번째 숫자 : 부여할 수 있는 최소레벨. 위의 경우 "0"을 지정할 수 없음
* 네번째 숫자 : 부팅시 출력될 로그레벨



#### 다음 명령으로 설정을 변경 할 수 있다.
```Shell
echo 7 4 1 7 > /proc/sys/kernel/printk
```

<br>

출처 : <http://ok2513.tistory.com/9>, <http://blog.naver.com/PostView.nhn?blogId=agnazz&logNo=100122290010&parentCategoryNo=&categoryNo=40&viewDate=&isShowPopularPosts=false&from=postView>

<br />

## [**Table of Contents**](../README.md)