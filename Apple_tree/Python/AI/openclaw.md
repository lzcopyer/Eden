# openclaw 部署教程

## 准备依赖

> openclaw 需要node >= 22

```Bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install node
```

## 部署

``` Bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

## 配置国内聊天软件

### 安装channel plugins

``` Bash
openclaw plugins install @openclaw-china/channels
```

### 配置channel plugins

``` Bash
openclaw config set channels.feishu '{
  "enabled": true,
  "appId": "cli_xxxxxx",
  "appSecret": "your-app-secret",
  "sendMarkdownAsCard": true

}' --json
```

## 参考链接

[https://docs.openclaw.ai/start/getting-started#3-5-quick-verify-2-min](docs.openclaw.ai)
[https://github.com/BytePioneer-AI/moltbot-china](openclaw-china)
[https://itho.cn/574.html](飞书机器人配置)
