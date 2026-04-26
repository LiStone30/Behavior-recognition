# 代码挂载到远程服务器

```bash

Remote-SSH: Connect to Host...

ilstone@ilstone-ms-7d36

```

# 不进入 ubuntu 的 容器，在 ubuntu上进行开发
## 构建一个开发镜像
python:3.12-slim-bookworm

podman pull docker.1ms.run/library/python:3.12-slim-bookworm

podman build -f dockerfile_dev -t python_dev:312 .

localhost/python_dev:312
## 运行临时开发容器
podman run -it --rm localhost/python_dev:312  bash

## 测试下 podman 容器的GPU支持
podman pull docker.1ms.run/nvidia/cuda:12.6.2-cudnn-devel-ubuntu24.04

podman run --rm -it --device nvidia.com/gpu=all docker.1ms.run/nvidia/cuda:12.6.2-cudnn-devel-ubuntu24.04 nvidia-smi

## 运行k8s长期开发容器 



# 先开始模型选型和部署吧

