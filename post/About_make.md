<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
Makefile에 대한 사항들을 정리하기 위한 페이지이다.

<br />

### Basic Example  
* 모듈을 빌드하는 간단한 예제이다.  
	```Makefile
	TARGET_NAME = ckun
	KERNEL_DIR  = /lib/modules/$(shell uname -r)/build
	PWD         = $(shell pwd)

	obj-m			= $(TARGET_NAME).o
	$(TARGET_NAME)-objs	= main.o \
				  init.o

	all:
		make -C $(KERNEL_DIR) M=$(PWD) modules

	install:
		make -C $(KERNEL_DIR) M=$(PWD) modules_install

	clean:
		make -C $(KERNEL_DIR) M=$(PWD) clean
	```
<br />

### modules_install  
* 기본적으로 /lib/modules/ 이하에 설치되며, `INSTALL_MOD_PATH` 를 사용하여 설치 경로를 변경할 수 있다.  
	```Makefile
	make -C $(KERNEL_DIR) M=$(PWD) INSTALL_MOD_PATH=/usr/local/ modules_install  
	```
* 빌드와 설치를 동시에 하기 위해서는 아래와 같이 작성한다.
	```Makefile
	make -C $(KERNEL_DIR) M=$(PWD) modules modules_install
	```
* 이렇게 설치하여야 `modprobe {module_name}`을 사용할 수 있다. 참고로 `modprobe -r`은 모듈 자체를 삭제하지는 않는다.
	```Shell
	shell> modprobe ckun.ko
	shell> modprobe -r ckun.ko
	```
<br />

### clean
* 생성된 파일을 삭제하기 위해 `rm` 명령어를 사용하기도 하는데 `make -C .... clean` 하면 소스 파일을 제외하고 전부 삭제해 준다
	```Makefile
	make -C $(KERNEL_DIR) M=$(PWD) clean
	```   

<br />

## [**Table of Contents**](../README.md)