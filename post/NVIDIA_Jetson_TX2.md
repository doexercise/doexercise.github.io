# **NVIDIA Jetson TX2**

> 이것은 `Jetson TX2 Developer Kit` 의 초기 설정에 대한 글이다.

&nbsp;

## **Updates**
- 2019.02.27 - init

&nbsp;

## **Jetpack Installation**
처음 전원을 켜면 ubuntu 설치 작업 지시가 나온다. 이대로 수행하면 익숙한 ubuntu desktop 화면을 볼 수 있다. 기본 계정은 nvidia, 비밀번호 역시 nvidia 이다.  
다만, 이 작업을 굳이 수행할 필요가 없다. Jetpack을 설치하면 어차피 OS 가 설치된다. 따라서 Jetpack 설치부터 시작한다.

Jetpack 의 설치 방법은 아래 링크에서 확인 할 수 있다. 이 링크를 참고하여 **<u>Jetpack 만 설치하자.</u>**
> https://judo0179.tistory.com/19

&nbsp;

## **Tensorflow Installation**
Jetpack Installation 항목에서 소개한 링크에 Tensorflow 설치 방법이 함께 나와 있으나 최신 버전을 설치해야 하는 것이 아니라면 굳이 그렇게 복잡한 절차를 거칠 필요가 없다. 참고로, 현재 기준으로 기본 설치되는 python 버전은 3.5 이다.  

설치는 간단하다. Jetpack 3.3 에서 아래와 같이 수행하면 Tensorflow 1.9 가 설치된다.
```shell
sudo apt-get install python3-numpy swig python3-dev python3-pip python3-wheel -y
sudo pip3 install --upgrade pip
sudo pip3 install --upgrade numpy
sudo pip3 install --extra-index-url=https://developer.download.nvidia.com/compute/redist/jp33 tensorflow-gpu
```

설치가 되었는지 확인하는 방법은 다음과 같다.
```python
nvidia@tegra-ubuntu:~$ python3
Python 3.5.2 (default, Nov 12 2018, 13:43:14)
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> tf.__version__
'1.9.0'
>>> quit()
```

&nbsp;

## **OpenCV 3.4.2 Installation**
설치에 관해 매우 잘 정리된 사이트가 있다. 아래 링크의 설명대로 진행하면 된다.  
> https://jkjung-avt.github.io/opencv3-on-tx2/  

다만, 몇가지 참고할 사항은 다음과 같다.
- `numpy` 는 tensorflow 설치시 업그레이드 하였으므로 위 링크에서 numpy 를 삭제/재설치 하는 부분은 생략해도 된다.
- 링크에는 3.4.0 을 기준으로 하고 있으나 3.4.2 도 잘 동작된다고 설명하고 있다. 3.4.2 를 설치하기 위해서는 OpenCV 소스를 `https://github.com/opencv/opencv/archive/3.4.2.zip` 에서 받아서 설치하면 된다.

&nbsp;

## **How to use Tensorflow-models**
Tensorflow 에서 제공하는 model 을 사용하기 위해서는 몇가지 절차가 필요하다. 아래 링크를 참조하라.
> https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/installation.md  

Tensorflow v1.9 와 호환되는 model 은 `https://github.com/tensorflow/models/archive/v1.9.0.zip` 에서 다운로드 할 수 있다.  

### 몇가지 문제점
- pip 를 이용하여 protobuf 를 설치하는 경우 `2.6.1` 이 설치된다. 이 버전은 사용할 수 없으므로 3.x 버전의 컴파일된 파일을 이용한다.  
    > https://github.com/protocolbuffers/protobuf/releases/download/v3.6.1/protoc-3.6.1-linux-aarch_64.zip


&nbsp;

## **Misc**
- how to enable all cpus on JetsonTX2
    ```shell
    sudo sh -c "echo 1 > /sys/devices/system/cpu/cpu1/online"
    sudo sh -c "echo 1 > /sys/devices/system/cpu/cpu2/online"
    ```

- how to run fan on JetsonTX2
    ```shell
    sudo ~/jetson_clocks.sh
    ```

- JetsonTX2 Performance mode  
    > https://www.jetsonhacks.com/2017/03/25/nvpmodel-nvidia-jetson-tx2-development-kit/    

- JetsonTX2 Factory Reset
OS를 완전히 초기화 하고 싶은 경우에 Jetpack 인스톨러를 실행하고(GUI) 패키지 선택 항목에서 `Target-JetsonTX2 -> Flash OS Image to Target` 항목의 `Action`을 `install` 로 바꾸어 진행한다. 그러면 새로 OS 이미지를 설치한다.  
