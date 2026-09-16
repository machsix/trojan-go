---
title: "使用API动态管理用户"
draft: false
weight: 10
---

### 注意，Trojan不支持这个特性

Trojan-Go使用gRPC提供了一组API，支持用户信息增删改查、流量统计、速度统计和IP连接数统计。Trojan-Go本身集成了API控制功能，也即可以使用一个Trojan-Go实例控制另一个Trojan-Go服务器。

需要在被控制服务端的配置中添加API设置，例如：

```json
{
    ...
    "api": {
        "enabled": true,
        "api_addr": "127.0.0.1",
        "api_port": 10000,
        "allow_payload_capture": false
    }
}
```

```allow_payload_capture```默认为```false```，生产环境应保持该值。它仅在使用```apidebug```构建标签编译的二进制中生效；默认发布版本会将```GetRecords```中```IncludePayload=true```的请求降级为只输出元数据，因此生产脚本不会意外泄露连接字节。使用```apidebug```构建且此项为```false```时，```GetRecords```会返回```PermissionDenied```，以明确提示配置错误。

当```api_addr```绑定到非回环地址时，如果没有启用TLS，或启用了TLS但```verify_client```为关闭状态，trojan-go会在启动时分别记录```WARN```。程序不会拒绝启动，以保留私有网络中的既有部署；但任何暴露到互联网的管理API均应使用mTLS保护，否则其中的明文gRPC和管理命令可被读取。

启动Trojan-Go服务器：

```shell
./trojan-go -config ./server.json
```

再使用另一个Trojan-Go连接服务器进行管理，基本命令格式为：

```shell
./trojan-go -api-addr SERVER_API_ADDRESS -api COMMAND
```

其中```SERVER_API_ADDRESS```为API地址和端口，例如```127.0.0.1:10000```。```COMMAND```可为：

- ```list```：列出所有用户
- ```get```：获取某个用户信息
- ```set```：设置用户信息（添加、删除或修改）

下面是一些例子。

1. 列出所有用户信息

    ```shell
    ./trojan-go -api-addr 127.0.0.1:10000 -api list
    ```

    返回JSON格式的用户信息，其中包括在线IP数、实时速度、总上传/下载流量等：

    ```json
    [{"user":{"hash":"d63dc919e201d7bc4c825630d2cf25fdc93d4b2f0d46706d29038d01"},"status":{"traffic_total":{"upload_traffic":36393,"download_traffic":186478},"speed_current":{"upload_speed":25210,"download_speed":72384},"speed_limit":{"upload_speed":5242880,"download_speed":5242880},"ip_limit":50,"quota":10737418240}}]
    ```

    流量单位均为字节。```quota```为流量配额：负数表示不限额，```0```表示禁用用户，正数表示字节限额。

2. 获取一个用户的信息

    可用```-target-password```指定密码，或用```-target-hash```指定目标用户密码的SHA224散列值：

    ```shell
    ./trojan-go -api-addr 127.0.0.1:10000 -api get -target-password password
    ```

    或：

    ```shell
    ./trojan-go -api-addr 127.0.0.1:10000 -api get -target-hash d63dc919e201d7bc4c825630d2cf25fdc93d4b2f0d46706d29038d01
    ```

    两条命令等价。返回格式与```list```类似：

    ```json
    {"user":{"hash":"d63dc919e201d7bc4c825630d2cf25fdc93d4b2f0d46706d29038d01"},"status":{"traffic_total":{"upload_traffic":36393,"download_traffic":186478},"speed_current":{"upload_speed":25210,"download_speed":72384},"speed_limit":{"upload_speed":5242880,"download_speed":5242880},"ip_limit":50,"quota":10737418240}}
    ```

3. 添加用户

    ```shell
    ./trojan-go -api-addr 127.0.0.1:10000 -api set -add-profile -target-password password
    ```

4. 删除用户

    ```shell
    ./trojan-go -api-addr 127.0.0.1:10000 -api set -delete-profile -target-password password
    ```

5. 修改用户信息

    ```shell
    ./trojan-go -api-addr 127.0.0.1:10000 -api set -modify-profile -target-password password \
        -ip-limit 3 \
        -upload-speed-limit 5242880 \
        -download-speed-limit 5242880 \
        -quota 10737418240
    ```

    此命令将密码为```password```的用户上传和下载速度限制为5 MiB/s，同时连接IP数限制为3，并将流量配额设置为10 GiB（10737418240字节）。速度单位为字节/秒；速度或IP数量为0或负数时表示不限制。

    CLI会区分“未传入参数”和“显式传入零值”。只有实际出现在命令行中的参数才会发送到服务端，未提及的字段会保持原值。例如```trojan-go -api set -modify-profile -target-password password -ip-limit 3```不会把该用户已有的配额清零；```quota```字段根本不会出现在请求中。

6. 管理用户配额

    可从```get```或```list```结果的```quota```字段读取配额。使用CLI设置或修改配额：

    ```shell
    ./trojan-go -api-addr 127.0.0.1:10000 -api set -modify-profile -target-password password -quota 10737418240
    ```

    该命令将配额设为10 GiB（10737418240字节）。语义如下：

    - **负数**（通常为```-1```）：不限额。新建用户默认值为```-1```，因此YAML/JSON配置中的静态```password```列表仍可认证。
    - **```0```**：禁用用户，认证将被拒绝，已有隧道也会通过该用户的截止信号关闭。
    - **正数**：字节限额。一旦```download + upload >= quota```，用户的活动隧道会在越过阈值后约一秒内关闭，而非等到下一次轮询。

    在线路协议中，proto3的```quota```字段为```optional int64```，因此调用方可区分“字段缺失”（保留原值）和“字段为```0```”（显式禁用）。添加用户时省略配额会使用默认```-1```；显式```quota=0```会禁用用户。修改时省略配额会保留旧值；显式```quota=0```会禁用用户。CLI用```flag.Visit```实施相同约定，仅在命令行传入```-quota```时发送该字段。下游Go gRPC使用者应重新生成桩代码：```UserStatus.Quota```现为```*int64```，并提供标准的```GetQuota()```访问器。
