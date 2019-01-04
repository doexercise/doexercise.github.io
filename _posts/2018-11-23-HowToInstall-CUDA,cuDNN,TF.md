# **How to benchmark GPUs using Tensorflow**

## **Environment**

- OS : ubuntu 16.04.5 (4.15.0-29-generic)
- gpu : nvidia gtx 1070 mxm
  - driver : 410.72
  - cuda : 9.0.176
  - cuDNN : cuda-9.0-linux-x64-v7.4.1.5
- python : 3.5.2
  - pip : 18.1
  - tensorflow : tf-nightly-gpu 1.13.0.dev20181122

&nbsp;

## **Disable nouveau**

```shell
#!/bin/bash
CONF=/etc/modprobe.d/blacklist-nouveau.conf

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
root@localhost# wget http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_9.0.176-1_amd64.deb
root@localhost# dpkg -i cuda-repo-ubuntu1604_9.0.176-1_amd64.deb
root@localhost# apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub
root@localhost# apt-get update
root@localhost# apt-get install cuda-9-0
```

Add envs to `~/.bashrc`

```shell
export PATH=$PATH:/usr/local/cuda/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64
```

&nbsp;

## **Install cuDNN-7.4**

Download `cudnn-9.0-linux-x64-v7.4.1.5.tgz` at [developer.nvidia.com](https://developer.nvidia.com/cudnn)  
And perform following commands

```shell
root@localhost# tar xfz cudnn-9.0-linux-x64-v7.4.1.5.tgz
root@localhost# cp -P cuda/include/* /usr/local/cuda/include/
root@localhost# cp -P cuda/lib64/* /usr/local/cuda/lib64/
```

&nbsp;

## **Install Tensorflow**

Do not install `tensorflow-gpu`, it is not compatible with `tf_cnn_benchmarks.py`

```shell
root@localhost# python3 -m pip install tf-nightly-gpu
```

&nbsp;

## **Tensorflow-Benchmarks**

Download scripts

```shell
root@localhost# wget https://github.com/tensorflow/benchmarks/archive/master.zip
```

Run scripts

```shell
#!/bin/bash

PY=/usr/bin/python3
SCRIPTS=/root/benchmarks-master/scripts/tf_cnn_benchmarks/tf_cnn_benchmarks.py

${PY} ${SCRIPTS} --num_gpus=1 --batch_size=32 --model=resnet50  --variable_update=parameter_server --local_parameter_device=cpu
```

> refer to [github](https://github.com/tensorflow/benchmarks)