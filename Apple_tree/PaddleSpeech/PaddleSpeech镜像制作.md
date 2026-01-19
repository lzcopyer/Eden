
# 准备Dockerfile

## 构建前准备

 下载`PaddleCloud`、`PaddleSpeech`源码

``` Dockerfile
ARG IMAGE_TAG=3.2.0
ARG PADDLE_TOOLKIT=PaddleSpeech
FROM paddlepaddle/paddle:${IMAGE_TAG}

# update apt cache, install ssh

RUN apt-get -y update && apt-get -y dist-upgrade

RUN mkdir -p /var/run/sshd /root/.ssh
RUN /usr/sbin/sshd -p 22; if [ $? -ne 0 ]; then \
    rm -rf /etc/ssh/sshd_config && apt-get install \
    -y openssh-server; fi

# disable UseDNS
RUN sed -i "s/#UseDNS .*/UseDNS no/" /etc/ssh/sshd_config
RUN sed -i -r "s/^(.*pam_nologin.so)/#\1/" /etc/pam.d/sshd
RUN ssh-keygen -A

COPY paddlenlp /root/.paddlenlp
COPY paddlespeech /root/.paddlespeech
COPY nltk_data /root/nltk_data

# add tini
# ENV TINI_VERSION v0.19.0
# ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
# RUN chmod +x /tini
COPY ./PaddleCloud/tekton/dockerfiles/Resource/tini /tini
RUN chmod +x /tini

ARG PADDLE_TOOLKIT
COPY ${PADDLE_TOOLKIT} ./${PADDLE_TOOLKIT}

# previous install

RUN pip3 install --no-cache-dir --upgrade pip 
RUN pip3 install --no-cache-dir --upgrade jupyter-server jupyterlab ipykernel ipython 

WORKDIR /home/${PADDLE_TOOLKIT}

# toolkit install
RUN pip3 install .
RUN pip uninstall -y aistudio-sdk && pip install aistudio-sdk==0.2.6 paddlespeech-ctcdecoders 

ENTRYPOINT ["/tini", "--"]
CMD ["paddlespeech_server", "start", "--config_file", "/home/PaddleSpeech/paddlespeech/server/conf/application.yaml"]

# ssh
EXPOSE 22    
# jupyter lab   
EXPOSE 8888
```

## 构建

``` Bash
docker build --tag banana:v1.0 .
```


# 使用docker-compose 管理镜像

**`yaml`文件如下**

``` yaml
# 定义服务
services:
  # 服务名称，建议与镜像名称保持一致，便于管理
  paddlespeech-server:
    # 指定要使用的镜像，确保你本地有这个镜像，或者可以在 Docker Hub 上找到
    image: banana:v2.0
    
    # 容器启动后，自动执行的命令
    # 这里我们使用 PaddleSpeech 提供的 `paddlespeech_server` 命令来启动 TTS 服务
    # --config_file: 指定配置文件路径，通常在容器内部的某个位置，你可以根据实际情况修改
    # --port: 指定服务监听的端口
    # --streaming_server_port: 指定流式服务监听的端口
    # --model_dir: 指定模型文件所在的目录
    # --use_gpu: 如果你的机器支持 GPU，可以设置为 true 来加速
    command: ["paddlespeech_server", "start", "--config_file", "./paddlespeech/server/conf/application.yaml", "--log_file", "/data/logs/paddlespeech_server.log"]

    # 端口映射，将宿主机的端口映射到容器内部的端口
    # 8090: TTS 服务端口
    # 8092: 流式 TTS 服务端口
    ports:
      - "8000:8090"
      - "8002:8092"
      
    # 数据卷挂载
    # 将宿主机的数据卷挂载到容器内部的指定路径
    volumes:
      - ../data:/data
      - ../conf:/home/PaddleSpeech/paddlespeech/server/conf
      
    # 重启策略，除非手动停止，否则容器异常退出时会自动重启
    restart: unless-stopped
```

**直接启动**

``` Bash
docker-compose -f dokcer-compose/compose.yaml up -d
```


# 提示

- 模型安装
	只需将模型挂载到容器内部即可
	[模型安装路径](https://github.com/PaddlePaddle/PaddleSpeech/blob/d1c70a780985e226c00a1d4c6e22626c4f668444/paddlespeech/resource/pretrained_models.py)