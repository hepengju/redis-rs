# redis-me

RedisME 使用的 [redis-rs](https://github.com/redis-rs/redis-rs) fork。只维护分支 `redis-me`。

Rust 包名是 `redis-me-rs`，库名仍是 `redis`，业务代码继续 `use redis::...`。crates.io 上的 `redis` 属于上游，不能用这个名字发布。

## 当前基线

上游标签 **`redis-1.8.1`**，提交 `d8863e6001bc94423135a2258311c65f2163c6ab`（2026-10-30，`Prepare version`）。

这是 1.7.0（`2a29a8f`，与标签 `redis-1.7.0` 相同）之后的 1.7.1、1.8.0、1.8.1。不要跟到 `2.0.0-rc`。

本地 `main` 指向这个标签。`redis-me` 上的提交都叠在它上面，树已经和标签对齐。不要用源码压缩包覆盖工作区，那会把测试夹具的换行改成 CRLF，并丢掉 `connection.rs`、`release.sh` 的可执行位。

## 相对上游的改动

只动连接和 TLS，方便 RedisME 的旧客户端证书、SSH 隧道和 HTTP/SOCKS 代理。

| 改动 | 行为 | 位置 |
|---|---|---|
| X.509 v1 客户端证书 | `tls-rustls` 下不再走 `with_client_auth_cert`（webpki 会拒绝 v1）。改为 `CertifiedKey::new` + `with_client_cert_resolver`，OpenSSL 1.0.2 默认签发的 PEM 仍能做 mTLS | `redis/src/connection.rs` |
| `ConnectionDialer` | 同步建连可换成自定义传输。`host` 保持地址里的原始字符串，不先做 `ToSocketAddrs`。未设置时与上游一致。Dialer 返回明文流，`rediss://` 仍由 redis-rs 在流上包 TLS | `connection.rs` 的 `ConnectionDialer` / `RedisStream`；`Client::set_dialer`；`ClusterClientBuilder::dialer`（每个集群节点共用）；`SentinelClientBuilder::set_dialer`（哨兵节点和发现出的主从都走它） |
| 哨兵 Insecure TLS 保留客户端证书 | `TlsMode::Insecure` 只跳过服务端校验，仍然附上 `TlsCertificates`。Redis 默认 `tls-auth-clients yes`，不带客户端证书会在握手阶段失败 | `redis/src/sentinel.rs` 的 `create_connection_info` |
| 适配 1.8 的 `Debug` | 1.8 为集群参数和哨兵 builder 写了穷尽拆解的 `Debug`。`ConnectionDialer` 没有 `Debug`，这里用「有没有设置」来打印，避免编译失败，也避免 dialer 在转换时被丢掉 | `cluster_handling/client.rs`、`sentinel.rs` |
| 发布名和许可证文件 | 包名 `redis-me-rs`，`[lib] name = "redis"`。`redis/LICENSE` 改成许可证正文。上游这里是指向仓库根目录的符号链接，在 Windows 上打包会变成字面量 `../LICENSE` | `redis/Cargo.toml`、`redis/LICENSE`、`Cargo.lock`、`redis-test` 与 `afl/parser` 的 path 依赖 |

RedisME 依赖：

```toml
redis = { package = "redis-me-rs", version = "1.8.1", features = [
  "acl", "cluster", "sentinel", "json", "vector-sets",
  "tls-rustls", "tls-rustls-insecure",
] }
```

本机 Cargo 把 crates.io 换成了 rsproxy，发布时要写明 registry：

```bash
cargo publish -p redis-me-rs --registry crates-io
```

## 下次同步

1. 从上游把新标签抓下来，例如 `redis-1.8.2`。先看 changelog，确认不是 2.0。
2. 把 `redis-me` 上「旧标签之后」的提交接到新标签上：

```bash
git rebase --onto redis-1.8.2 redis-1.8.1 redis-me
```

3. 冲突优先看 `redis/src/connection.rs`、`redis/src/sentinel.rs`、`redis/src/cluster_handling/client.rs`。上游的 TLS、哨兵和 `Debug` 改动留下；上表里的 dialer、v1 证书解析器、Insecure TLS 仍附带客户端证书也留下。
4. 编译并跑 fork 自己的单测：

```bash
cargo test -p redis-me-rs --features "acl,cluster,sentinel,json,vector-sets,tls-rustls,tls-rustls-insecure" --lib -- connection_dialer connection_info_debug get_connection_info_copies_dialer sentinel_set_dialer sentinel_insecure
```

5. 若只改了 fork、上游还没有新版本，把 `redis/Cargo.toml` 的版本号往上加一档再发布。版本号属于 `redis-me-rs`，和官方 `redis` 互不影响。
