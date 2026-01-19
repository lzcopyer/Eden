# 构建镜像

## 编译准备

准备`Dockerfile`以及相关的模块包
参考[PaddleCloud/dockerfiles](https://github.com/PaddlePaddle/PaddleCloud/blob/main/tekton/dockerfiles)
提示：可以在[Paddle镜像库](https://hub.docker.com/u/paddlepaddle)查看最新的`Paddle`镜像版本

``` Dockerfile
FROM paddlepaddle/paddle:2.3.0

RUN rm -rf /etc/apt/* && mkdir -p /etc/apt/apt.conf.d \
    /Resource/trusted.gpg.d /etc/apt/preferences.d
COPY ./Resource/sources.list /etc/apt/
COPY ./Resource/trusted.gpg /etc/apt/
COPY ./Resource/trusted.gpg.d /etc/apt/trusted.gpg.d

# 更新apt缓存、安装ssh服务
RUN apt-get update && apt-get dist-upgrade
RUN /usr/sbin/sshd -p 22; if [ $? -ne 0 ]; then \
apt-get install -y openssh-server; fi
RUN mkdir -p /var/run/sshd /root/.ssh
# 不使用DNS
RUN sed -i "s/#UseDNS .*/UseDNS no/" /etc/ssh/sshd_config
RUN sed -i -r "s/^(.*pam_nologin.so)/#\1/" /etc/pam.d/sshd
RUN ssh-keygen -A

COPY PaddleSlim ./PaddleSlim
COPY PaddleOCR ./PaddleOCR
COPY PaddleNLP ./PaddleNLP
COPY PaddleDetection ./PaddleDetection
COPY PaddleSeg ./PaddleSeg
COPY PaddleClas ./PaddleClas
COPY PaddleSpeech ./PaddleSpeech
COPY PaddleRec ./PaddleRec
COPY PaddleHelix ./PaddleHelix
COPY PaddleScience ./PaddleScience
COPY paddlenlp_module /root/.paddlenlp/models
COPY paddlespeech_module /root/.paddlespeech/models
COPY nltk_data.tar.gz /root/nltk_data.tar.gz

RUN apt-get install -y libsndfile1
RUN pip3.7 install --no-cache-dir --upgrade pip \
    -i https://pypi.tuna.tsinghua.edu.cn/simple/
RUN pip3.7 install --no-cache-dir --upgrade jupyter jupyterlab \
    ipykernel ipython -i https://pypi.tuna.tsinghua.edu.cn/simple

WORKDIR /home/PaddleSlim
RUN pip3.7 install --no-cache-dir -r requirements.txt \
    -i https://pypi.tuna.tsinghua.edu.cn/simple
RUN python3.7 setup.py install

WORKDIR /home/PaddleOCR
RUN pip3.7 install --no-cache-dir -r requirements.txt \
    -i https://pypi.tuna.tsinghua.edu.cn/simple
RUN python3.7 setup.py install

WORKDIR /home/PaddleNLP
RUN pip3.7 install --no-cache-dir visualdl regex pybind11 zstandard \
    -i https://pypi.tuna.tsinghua.edu.cn/simple
RUN pip3.7 install --no-cache-dir -r requirements.txt \
    -i https://pypi.tuna.tsinghua.edu.cn/simple
RUN python3.7 setup.py install

WORKDIR /home/PaddleDetection
RUN pip3.7 install --no-cache-dir -r requirements.txt \
    -i https://pypi.tuna.tsinghua.edu.cn/simple
RUN python3.7 setup.py install

WORKDIR /home/PaddleSeg
RUN pip3.7 install --no-cache-dir -r requirements.txt \
    -i https://pypi.tuna.tsinghua.edu.cn/simple
RUN python3.7 setup.py install

WORKDIR /home/PaddleClas
RUN pip3.7 install --no-cache-dir -r requirements.txt \
    -i https://pypi.tuna.tsinghua.edu.cn/simple

WORKDIR /home/PaddleSpeech
RUN pip3.7 install . -i https://pypi.tuna.tsinghua.edu.cn/simple

WORKDIR /home

# ssh
EXPOSE 22    
# jupyter lab   
EXPOSE 8888

CMD ["sleep", "infinity"]
```

## 编译

``` Bash
cd $build
docker build .
```

# 使用docker-compose部署

## 编排YAML文件

``` YAML
version: '3.8'

# 定义数据卷，用于持久化数据，避免容器删除后数据丢失
volumes:
  data: "../data"
  conf: "../conf"

# 定义网络，如果需要多个容器之间通信，可以创建一个自定义网络
networks:
  paddle-network:
    external: true

# 定义服务
services:
  # 服务名称，建议与镜像名称保持一致，便于管理
  paddlespeech-server:
    # 指定要使用的镜像，确保你本地有这个镜像，或者可以在 Docker Hub 上找到
    image: paddlespeech:v1.0
    
    # 容器启动后，自动执行的命令
    # 这里我们使用 PaddleSpeech 提供的 `paddlespeech_server` 命令来启动 TTS 服务
    # --config_file: 指定配置文件路径，通常在容器内部的某个位置，你可以根据实际情况修改
    # --port: 指定服务监听的端口
    # --streaming_server_port: 指定流式服务监听的端口
    # --model_dir: 指定模型文件所在的目录
    # --use_gpu: 如果你的机器支持 GPU，可以设置为 true 来加速
    command: ["paddlespeech_server", "start", "--config_file", "./paddlespeech/server/conf/application.yaml"]

    # 端口映射，将宿主机的端口映射到容器内部的端口
    # 8090: TTS 服务端口
    # 8092: 流式 TTS 服务端口
    ports:
      - "8090:8090"
      - "8092:8092"
      
    # 网络设置
    networks:
      - paddle-network
      
    # 数据卷挂载
    # 将宿主机的数据卷挂载到容器内部的指定路径
    volumes:
      - data:/data
      - conf:/home/PaddleSpeech/paddlespeech/demos/streaming_tts_server/conf
      
    # 重启策略，除非手动停止，否则容器异常退出时会自动重启
    restart: unless-stopped
```

## 启动！

``` Bash
docker-compose -f docker-compose.yaml up -d
```