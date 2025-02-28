# 本地部署相关

### 1. 本地部署初始化后，密码错误如何重置？

若使用 docker compose 方式部署，可执行以下命令进行重置

```
docker exec -it docker-api-1 flask reset-password
```

输入账户 email 以及两次新密码即可。

### 2. 本地部署日志中报 File not found 错误，如何解决？

```
ERROR:root:Unknown Error in completion
Traceback (most recent call last):
  File "/www/wwwroot/dify/dify/api/libs/rsa.py", line 45, in decrypt
    private_key = storage.load(filepath)
  File "/www/wwwroot/dify/dify/api/extensions/ext_storage.py", line 65, in load
    raise FileNotFoundError("File not found")
FileNotFoundError: File not found
```

该错误可能是由于更换了部署方式，或者 `api/storage/privkeys` 删除导致，这个文件是用来加密大模型密钥的，因此丢失后不可逆。可以使用如下命令进行重置加密公私钥：

*   Docker compose 部署

    ```
    docker exec -it docker-api-1 flask reset-encrypt-key-pair
    ```
*   源代码启动

    进入 api 目录

    ```
    flask reset-encrypt-key-pair
    ```

    按照提示进行重置。

### **3. 安装时后无法登录，登录成功，但后续接口均提示 401？**

这可能是由于切换了域名/网址，导致前端和服务端跨域。跨域和身份会涉及到两方面的配置：

1. CORS 跨域配置
   1.  `CONSOLE_CORS_ALLOW_ORIGINS`

       控制台 CORS 跨域策略，默认为 `*`，即所有域名均可访问。
   2.  `WEB_API_CORS_ALLOW_ORIGINS`

       WebAPP CORS 跨域策略，默认为 `*`，即所有域名均可访问。
2.  COOKIE 策略配置

    Cookie 策略分为三个配置 `HttpOnly`、`SameSite` 和 `Secure`。

    1. `HttpOnly`：默认为 true，正常不需要修改，用于防止 XSS 攻击，即 JS 无法获取 Cookie 内容，只能在 Http 请求中带上。
    2. `SameSite`：分为三档，Strict、Lax 和 None，而由于 Dify 需要在 Github、Google 外部域名授权回调时能够从 Cookie 获取身份信息，因此只能在 Lax 和 None 之间选择，其中 None 完全可以跨域访问。
    3. `Secure`：该参数限制是否服务端接口必须在 https 时下才可将 Cookie 存到本地，该参数在跨域时必须为 true（本地 localhost / 127.0.0.1 不同端口除外），否则浏览器不予通过。

#### 推荐配置

根据上述配置说明，我们推荐这三种场景下的配置：

*   本地调试（默认策略）

    开发模式同域策略。 支持 HTTP/HTTPS 协议，但需要保证前端页面和接口同域。

    <pre><code><strong>WEB_API_CORS_ALLOW_ORIGINS: '*'
    </strong>CONSOLE_CORS_ALLOW_ORIGINS: '*'
    COOKIE_HTTPONLY: 'true'
    COOKIE_SAMESITE: 'Lax'
    COOKIE_SECURE: 'false'
    </code></pre>
*   跨域策略（请勿应在生产）

    服务端与 web 客户端跨域，服务端必须为 https。 由于 SameSite=None 必须配合 Secure=true，因此服务端必须为 `https` 协议才能实现跨域访问，可以用在服务端在远程并且提供 `https` 协议支持，或者本地单独启动服务端和前端项目（localhost，但不同端口，实测可用，虽然提示 warning）。

    ```
    WEB_API_CORS_ALLOW_ORIGINS: 'https://your-domain-for-web-app'
    CONSOLE_CORS_ALLOW_ORIGINS: 'https://your-domain-for-console'
    COOKIE_HTTPONLY: 'true'
    COOKIE_SAMESITE: 'None'
    COOKIE_SECURE: 'true'
    ```
*   生产策略

    严格模式。 由于部分第三方集成需要支持回调并带着 cookie 信息，因此不能使用最高的 Strict 策略，因此需要严格限制 CORS 域名，以及设置 cookie 策略为 SameSite=Lax, Secure=true。

    ```
    WEB_API_CORS_ALLOW_ORIGINS: 'https://your-domain-for-web-app'
    CONSOLE_CORS_ALLOW_ORIGINS: 'https://your-domain-for-console'
    COOKIE_HTTPONLY: 'true'
    COOKIE_SAMESITE: 'Lax'
    COOKIE_SECURE: 'true'
    ```

#### 不可用场景

在前后端跨域，且服务端为 http 协议时，无任何 Cookie 策略可以支持该场景，请调整后端为 HTTPS 协议或者设置为同域。

### **4. 启动后页面一直在 loading，查看请求提示 CORS 错误？**

这可能是由于切换了域名/网址，导致前端和服务端跨域，请将 `docker-compose.yml` 中所有的以下配置项改为新的域名：

`CONSOLE_API_URL:` 控制台 API 的后端 URL。 `CONSOLE_WEB_URL:` 控制台网页的前端 URL。 `SERVICE_API_URL:` 服务 API 的 URL。 `APP_API_URL:` WebApp API 的后端 URL。 `APP_WEB_URL:` WebApp 的 URL。

更多信息，请查看：[环境变量](../../getting-started/install-self-hosted/environments.md)

### 5. 部署后如何升级版本？

如果你是通过镜像启动，请重新拉取最新镜像完成升级。 如果你是通过源码启动，请拉取最新代码，然后启动，完成升级。

源码部署更新时，需要进入 api 目录下，执行以下命令将数据库结构迁移至最新版本：

`flask db upgrade`

### 6. 使用 Notion 导入时如何配置环境变量？

[**Notion 的集成配置地址**](https://www.notion.so/my-integrations)\*\*。\*\*进行私有化部署时，请设置以下配置：

1. **`NOTION_INTEGRATION_TYPE`** ：该值应配置为（**public/internal**）。由于 Notion 的 Oauth 重定向地址仅支持 https，如果在本地部署，请使用 Notion 的内部集成。
2. **`NOTION_CLIENT_SECRET`** ： Notion OAuth 客户端密钥（用于公共集成类型）。
3. **`NOTION_CLIENT_ID`** ： OAuth 客户端ID（用于公共集成类型）。
4. **`NOTION_INTERNAL_SECRET`** ： Notion 内部集成密钥，如果 `NOTION_INTEGRATION_TYPE` 的值为 **internal**，则需要配置此变量。

### 7. 本地部署版，如何更改空间的名称？

在数据库 `tenants` 表里修改。

### 8. 想修改访问应用的域名，在哪里修改？

在 `docker_compose.yaml` 里面找到 APP\_WEB\_URL 配置域名。

### 9. 如果发生数据库迁移，需要备份哪些东西？

需要备份数据库、配置的存储以及向量数据库数据，若为 docker compose 方式部署，可直接备份 `dify/docker/volumes` 目录下所有数据内容。

### 10. 为什么 Docker 部署 Dify，本地启动 OpenLLM 用 127.0.0.1 却无法访问本地的端口？

127.0.0.1 是容器内部地址， Dify 配置的服务器地址需要宿主机局域网 IP 地址。

### 11. 本地部署版如何解决数据集文档上传的大小限制和数量限制。

可参考官网[环境变量说明文档](https://docs.dify.ai/v/zh-hans/getting-started/install-self-hosted/environments)去配置。

### 12. 本地部署版如何通过邮箱邀请成员？

本地部署版，邀请成员可通过邮箱邀请，输入邮箱邀请后，页面显示邀请链接，复制邀请链接转发给用户，用户打开链接通过邮箱登录设置密码即可登录到你的空间内。

### 13. 本地部署版本遇到这个错误需要怎么办 Can't load tokenizer for 'gpt2'

```
Can't load tokenizer for 'gpt2'. If you were trying to load it from 'https://huggingface.co/models', make sure you don't have a local directory with the same name. Otherwise, make sure 'gpt2' is the correct path to a directory containing all relevant files for a GPT2TokenizerFast tokenizer.
```

可参考官网[环境变量说明文档](https://docs.dify.ai/v/zh-hans/getting-started/install-self-hosted/environments)去配置。以及相关 [Issue](https://github.com/langgenius/dify/issues/1261)。

### 14. 本地部署 80 端口被占用应该如何解决？

本地部署 80 端口被占用，可通过停止占用 80 端口的服务，或者修改 docker-compose.yaml 里面的端口映射，将 80 端口映射到其他端口。通常 Apache 和 Nginx 会占用这个端口，可通过停止这两个服务来解决。

### 15. 文本转语音遇到这个错误怎么办？

```
[openai] Error: ffmpeg is not installed
```

由于 OpenAI TTS 实现了音频流分段，源码部署时需要安装 ffmpeg 才可正常使用，详细步骤：

**Windows:**

1. 访问 [FFmpeg 官方网站](https://ffmpeg.org/download.html)，下载已经编译好的 Windows shared 库。
2. 下载并解压 FFmpeg 文件夹，它会生成一个类似于 "ffmpeg-20200715-51db0a4-win64-static" 的文件夹。
3. 将解压后的文件夹移动到你想要的位置，例如 C:\Program Files\。
4. 将 FFmpeg 的 bin 目录所在的绝对路径添加到系统环境变量中。
5. 打开命令提示符，输入"ffmpeg -version"，如果能看到 FFmpeg 的版本信息，那么说明安装成功。

**Ubuntu:**

1. 打开终端。
2. 输入以下命令来安装 FFmpeg：`sudo apt-get update`，然后输入`sudo apt-get install ffmpeg`。
3. 输入"ffmpeg -version" 来检查是否安装成功。

**CentOS:**

1. 首先，你需要启用EPEL存储库。在终端中输入：`sudo yum install epel-release`
2. 然后，输入：`sudo rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm`
3. 更新 yum 包，输入：`sudo yum update`
4. 最后，安装 FFmpeg，输入：`sudo yum install ffmpeg ffmpeg-devel`
5. 输入"ffmpeg -version" 来检查是否安装成功。

**Mac OS X:**

1. 打开终端。
2. 如果你还没有安装 Homebrew，你可以通过在终端中输入以下命令来安装：`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
3. 使用 Homebrew 安装 FFmpeg，输入：`brew install ffmpeg`
4. 输入 "ffmpeg -version" 来检查是否安装成功。

### 16. 本地部署时，如果遇到 Nginx 配置文件挂载失败，如何解决？

```
Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: error mounting "/run/desktop/mnt/host/d/Documents/docker/nginx/nginx.conf" to rootfs at "/etc/nginx/nginx.conf": mount /run/desktop/mnt/host/d/Documents/docker/nginx/nginx.conf:/etc/nginx/nginx.conf (via /proc/self/fd/9), flags: 0x5000: not a directory: unknown: Are you trying to mount a directory onto a file (or vice-versa)? Check if the specified host path exists and is the expected type
```

请下载完整的项目，进入 docker 执行 `docker-compose up -d` 即可。

```
git clone https://github.com/langgenius/dify.git
cd dify/docker
docker compose up -d
```
