<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
Visual C/C++ 에서의 구조적 예외처리

<br />

## What is SEH?
try-except 문은 정상적으로 프로그램 실행을 종료시키는 이벤트가 발생할 때, 대상 응용 프로그램이 제어할 수 있도록 하는 C와 C++언어의 Microsoft 확장이며, 이를 구조적 예외처리라고 함.

## SEH vs try-catch  
try-except 문은 C, C++ 모두 사용가능하지만 C++의 경우 언어자체가 지원하고 있는 try-catch 를 사용하는 것이 보다 유연하므로 try-catch 를 사용하길 권장  

## Basic Example
* __try __except  
	```
	__try {
	     // 일반 코드
	}
	__except( EXCEPTION_FILTER ) {
	     // 예외처리 코드
	}
	```

* __try __finally  
	```
	__try {
	     // 일반 코드
	     if (error) {
	          __leave;     // => __try 블럭 맨 끝으로 이동
	     }
	}
	__finally {
	     // 무조건 마지막에 실행
	}
	```

* __try __except __finally 는 안됨. 단, 각각 중첩 사용은 가능
	```
	// 반대의 경우도 가능하며, 실행 순서가 달라질 수 있음에 유의!!
	__try {
	     __try {
	          // 일반 코드
	     }
	     __except( EXCEPTION_FILTER ) {
	          // 예외처리 코드
	     }
	}
	__finally {
	     // 무조건 실행할 코드
	}
	```

## EXCEPTION_FILTER  
* EXCEPTION FILTER 로 다음 값 사용  

	| 매크로 | 정수값 | 의미 |
	|:---|:---:|:---|
	| EXCEPTION_CONTINUE_EXECUTION | -1 | 예외를 무시하고 이후 계속 실행 |
	| EXCEPTION_CONTINUE_SEARCH | 0 | 현재 exception 블록 안에서 처리하지 않고 상위 핸들러에게 넘김 |
	| EXCEPTION_EXECUTE_HANDLER | 1 | 예외처리 수행 |

* 위의 값(-1, 0, 1) 을 전달하기 위해 함수호출, 조건식, 쉼표를 사용할 수 있음
	```
	/**
	 * 조건식의 경우
	 */

	void test_function(void)
	{
	     __try {
	          // Exception !!!
	     }
	     __except ( GetExceptionCode() == EXCEPTION_ACCESS_VIOLATION ? EXCEPTION_EXECUTE_HANDLER 	: EXCEPTION_CONTINUE_SEARCH ) {
	          // 예외처리 코드
	     }
	}

	/**
	 * 쉼표 사용의 예
	 */

	__except( nCode = GetExceptionCode(), nCode == STATUS_INTEGER_OVERFLOW ) {} // 뒷 문장의 	조건식 판별하여 0 또는 1 반환

	__except( printf("Exception!!!"), EXCEPTION_EXECUTE_HANDLER ) {}
	```

## 디바이스 드라이버의 경우
* 예외처리가 정상적으로 수행되면 BSOD를 피할 수 있다
* 다만, 모든 예외를 처리할 수 있는 것은 아니다.
* 예외처리에 오버헤드가 있으므로 성능 저하가 생길 수 있다.

<br />

## [**Table of Contents**](../README.md)
