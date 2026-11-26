# Caddy 配置示例

本文提供可复制修改的 Caddyfile 示例，包含 TLS SNI 分流、反向代理、WebDAV 和静态文件服务。

## 使用前须知

- TLS SNI 分流需要包含 `layer4` 模块的 Caddy 构建，通常需通过 `xcaddy` 集成 `caddy-l4`。
- WebDAV 示例需要包含提供 `webdav` 指令的模块；该指令不属于 Caddy 核心功能。
- 将示例中的域名、邮箱、上游地址、目录和凭据替换为实际值。
- 使用真实域名自动申请证书时，域名 DNS 应指向服务器，并确保相关验证端口可访问。
- 部署前先执行配置检查，再加载配置。

## 1. TLS SNI 分流

将 `intel.com` 和 `www.intel.com` 的 TLS 流量转发到 `127.0.0.1:10001`，其余 TLS 流量转发到 Caddy 的 HTTPS 服务。

```caddyfile
{
    email admin@example.com
    http_port 80
    https_port 1443

    layer4 {
        :443 {
            @intel tls sni intel.com www.intel.com
            route @intel {
                proxy 127.0.0.1:10001
            }

            route {
                proxy 127.0.0.1:1443
            }
        }
    }
}

https://a.example.com:1443 {
    reverse_proxy 127.0.0.1:3000
}

https://b.example.com:1443 {
    respond "OK" 200
}
```

说明：

- `:443` 接收外部 TLS 连接，`tls sni` 根据客户端 TLS 握手中的 SNI 主机名匹配流量。
- `proxy 127.0.0.1:10001` 是四层转发，不会解密 TLS；目标服务需要自行处理 TLS。
- 未匹配的连接转发到 `127.0.0.1:8443`，再由 Caddy 处理 HTTPS 请求。
- `https_port 8443` 设置 Caddy 的 HTTPS 服务端口。请确认端口未被占用，并按实际网络环境配置防火墙和端口转发。
- 如果目标服务需要特定证书或 SNI，应在目标服务上配置。

## 2. TLS SNI 分流：本地证书配置

此配置适合本地测试或内网环境。`local_certs` 使用 Caddy 本地 CA 签发证书；客户端需信任该 CA，才能避免证书信任警告。

```caddyfile
{
    admin off
    skip_install_trust
    local_certs
    pki {
        ca local {
            intermediate_lifetime 730d
        }
    }
    default_sni api.example.com
    log default {
        level ERROR
    }
    servers {
        protocols h1 h2c
    }
    http_port 80
    https_port 8443
    layer4 {
        :443 {
            @intel tls sni intel.com www.intel.com
            route @intel {
                proxy 127.0.0.1:10001
            }

            route {
                proxy 127.0.0.1:1443
            }
        }
    }
}

https://api.example.com:1443 {
    tls {
        issuer internal {
            lifetime 180d
        }
    }
    respond "503 Service Unavailable" 503 {
        close
    }
}

http://:81 {
    root * /var/www/html
    file_server
}
```

说明：

- `admin off` 关闭管理 API；关闭后无法通过管理 API 动态加载配置。
- `skip_install_trust` 阻止自动将本地 CA 安装到系统信任库。
- `default_sni` 为未提供 SNI 的 TLS 客户端指定默认站点名称；仅在确有需要时配置。
- `protocols h1 h2c` 只启用 HTTP/1.1 和明文 HTTP/2，请按应用需求选择。
- `http://:81` 在 81 端口提供静态文件。不要将敏感目录设置为站点根目录。

## 3. 基本反向代理

将 `app.example.com` 的请求转发到本机 `3000` 端口的应用：

```caddyfile
app.example.com {
    reverse_proxy 127.0.0.1:3000
}
```

通常无需手动设置 `Host` 请求头。如果上游明确要求特定主机名，可覆盖该请求头：

```caddyfile
app.example.com {
    reverse_proxy 127.0.0.1:3000 {
        header_up Host app.internal
    }
}
```

### 多个上游

```caddyfile
app.example.com {
    reverse_proxy 127.0.0.1:3000 127.0.0.1:3001
}
```

## 4. WebDAV 文件服务

此示例将 `/dav/*` 请求交给 WebDAV 服务，文件存放在 `/srv/webdav`。请确认 Caddy 构建包含 `webdav` 指令。

```caddyfile
dav.example.com {
    route /dav/* {
        basicauth {
            davuser $2a$14$示例哈希值请替换为真实生成结果
        }

        root * /srv/webdav
        webdav
    }
}
```

使用 Caddy 生成密码哈希：

```sh
caddy hash-password --plaintext '请替换为你自己的强密码'
```

将命令输出的完整哈希值复制到 `basicauth` 中，替换示例内容。不要将明文密码直接写入配置文件。

说明：

- 将 `dav.example.com` 替换为实际域名，并确保 DNS 指向服务器。
- 确认 Caddy 运行用户对 `/srv/webdav` 有所需的读写权限。
- WebDAV 基本认证应通过 HTTPS 使用。
- 部署前确认所安装的 WebDAV 模块支持上述指令和语法。

## 5. 静态文件服务

```caddyfile
files.example.com {
    root * /var/www/html
    file_server
}
```

## 6. 检查与加载配置

```sh
caddy validate --config /etc/caddy/Caddyfile
caddy reload --config /etc/caddy/Caddyfile
```
