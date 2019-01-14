# **How to benchmark GPUs using Tensorflow**

## **Environment**

- OS : ubuntu 16.04.5 (4.15.0-29-generic)
- gpu : nvidia gtx 1070 mxm
  - driver : 410.72
  - cuda : 9.0.176
  - cuDNN : cuda-9.0-linux-x64-v7.4.1.5
- python : 3.5.2
  - pip : 18.1
  - tensorflow : tf-nightly-gpu 1.12.0.dev20181012

&nbsp;

## **Disable nouveau**

```shell
#!/bin/bash
CONF=/etc/modprobe.d/blacklist-nouveau.conf
touch $CONF
echo "blacklist nouveau" >> $CONF
echo "options nouveau modeset=0" >> $CONF
update-initramfs -u
reboot
```

> From [askubuntu.com](https://askubuntu.com/questions/841876/how-to-disable-nouveau-kernel-driver)

&nbsp;

## **Install cuda-9.0**

**`cuda`** include the nvidia driver, so you do not need to install driver separately.

```shell
wget http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_9.0.176-1_amd64.deb
dpkg -i cuda-repo-ubuntu1604_9.0.176-1_amd64.deb
apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub
apt-get update
apt-get install cuda-9-0
```

Add envs to `~/.bashrc`

```shell
export PATH=$PATH:/usr/local/cuda/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64
```

&nbsp;

## **Install cuDNN-7.4**

Download `cudnn-9.0-linux-x64-v7.4.1.5.tgz` at [developer.nvidia.com](https://developer.nvidia.com/cudnn)  
And perform following commnds

```shell
tar xfz cudnn-9.0-linux-x64-v7.4.1.5.tgz
cp -P cuda/include/* /usr/local/cuda/include/
cp -P cuda/lib64/* /usr/local/cuda/lib64/
```

&nbsp;

## **Install Tensorflow**

Do not install `tensorflow-gpu`, it is not compatible with `tf_cnn_benchmarks.py`

```shell
python3 -m pip install tf-nightly-gpu==1.12.0.dev20181012
```

&nbsp;

## **Tensorflow-Benchmarks**

Download scripts

```shell
wget https://github.com/tensorflow/benchmarks/archive/cnn_tf_v1.12_compatible.zip
unzip cnn_tf_v1.12_compatible.zip
```

Run scripts

```shell
#!/bin/bash

PY=/usr/bin/python3
SCRIPTS=/root/benchmarks-cnn_tf_v1.12_compatible/scripts/tf_cnn_benchmarks/tf_cnn_benchmarks.py

${PY} ${SCRIPTS} --num_gpus=1 --num_batches=1000 --batch_size=32 --model=resnet50 \
                 --variable_update=parameter_server --local_parameter_device=cpu
```

> refer to [tensorflow github](https://github.com/tensorflow/benchmarks)

&nbsp;

---
## **This is installation guide in windows**

1. Install cuda-9.0  
	**Please don't install display driver included in cuda, that driver is not compatible with Aetina MXM.**
	- base installer : `cuda_9.0.176_win10_network.exe`
	- 4 patches : `cuda_9.0.176.1_windows.exe` ~ `cuda_9.0.176.4_windows.exe`

2. Install cuDNN 7.4.2.24  
	**Make sure the cuDNN version is compatible with cuda.**
	- unzip `cudnn-9.0-windows10-x64-v7.4.2.24.zip`
	- copy entire files and folders into cuda installation path
	  usually cuda installation path is `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v9.0`

3. Install python
	- `python-3.6.5-amd64.exe`

4. Install tf-nightly-gpu  
	**You don't need to install tensorflow-gpu seperatly.** 
	- run cmd.exe as administrator  
        ```shell
	  py -m pip install tf-nightly-gpu==1.12.0.dev20181010
        ```

5. Install benchmarks scripts  
	**The scripts and tf-nightly-gpu should be paired.**
	- unzip `benchmarks-cnn_tf_v1.12_compatible.zip` and remember where `tf_cnn_benchmarks.py` is
	- modify `%PYTHON%` and `%SCRIPTS%` variables in `run.bat`

&nbsp;

## **Troubleshooting**
1. Fail to install `Visual Studio Integration` in cuda installation progress.
	- When I was trying to install cuda_9.0, this error occured
	- To avoid, 
		1) Uncheck `Visual Studio Integration 9.0` in CUDA Components
		2) Or install cuda_9.2 first, and then install cuda_9.0.