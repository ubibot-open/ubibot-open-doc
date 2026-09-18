# 固件：新增串口配网指令

*[English](06-firmware-add-serial-command.md)*

`main/provisioning.c` 实现了协议 §1.2 目前定义的三个指令——`SetupWifi`、`SetupServer`、
`SetupDevice`——新加一个指令走的是完全一样的路子。这一章用 `SetupDevice`（三个里最简单的，只有
一个字符串字段）当模板走一遍。

## 每个指令都遵循的形状

每个指令都是一小组固定搭配的部件，全都在 `provisioning.c` 里：

```c
// main/provisioning.c
#define PROV_KEY_SN "sn"                    // 1. 一个 NVS key
#define PROV_SN_MAX_LEN 65                  // 2. 它的缓冲区最大长度
static char s_sn[PROV_SN_MAX_LEN];          // 3. 常驻内存里的当前生效值

// 4. 存到 NVS 里
static esp_err_t prov_save_sn(const char *sn)
{
  nvs_handle_t h;
  esp_err_t err = nvs_open(PROV_NVS_NAMESPACE, NVS_READWRITE, &h);
  if (err != ESP_OK) return err;
  err = nvs_set_str(h, PROV_KEY_SN, sn);
  if (err == ESP_OK) err = nvs_commit(h);
  nvs_close(h);
  return err;
}

// 5. 校验、保存、更新内存里的值、应答
static void prov_handle_setup_device(const cJSON *root)
{
  const cJSON *sn = cJSON_GetObjectItemCaseSensitive(root, "sn");

  if (!cJSON_IsString(sn) || sn->valuestring[0] == '\0')
  {
    printf("{\"c\":1,\"msg\":\"sn is required\"}\r\n");
    return;
  }
  if (strlen(sn->valuestring) >= sizeof(s_sn))
  {
    printf("{\"c\":1,\"msg\":\"sn too long\"}\r\n");
    return;
  }

  if (prov_save_sn(sn->valuestring) != ESP_OK)
  {
    printf("{\"c\":2,\"msg\":\"failed to save sn\"}\r\n");
    return;
  }

  strlcpy(s_sn, sn->valuestring, sizeof(s_sn));
  ESP_LOGI(TAG, "device provisioned: sn=%s", s_sn);
  printf("{\"c\":0,\"msg\":\"sn saved\"}\r\n");
}
```

每个字段都是同一套处理：必填字段缺失/为空/太长就用 `c:1` 加一句具体的错误信息拒绝；保存失败就
报 `c:2`；成功的话，NVS 和内存里的值**都**要更新（这样改动才能立刻应用到设备当前这次运行，不是
只在下次开机才生效），然后用 `c:0` 应答。

## 怎么把一个新指令接进去

1. **接到分发逻辑里**——`prov_handle_line()` 就是一串对 `command` 字段做 `strcmp` 的判断链；把你
   的指令名字加进去：

   ```c
   // main/provisioning.c，在 prov_handle_line() 里
   else if (strcmp(command->valuestring, "SetupDevice") == 0)
   {
     prov_handle_setup_device(root);
   }
   // 你的新指令的 else-if 加在这里，写法一样
   ```

2. **在 `Provision_Init()` 里加上"默认值+NVS 覆盖"的加载逻辑**——每个字段都是同样的两步：先从
   Kconfig 编译期默认值起步，然后检查 NVS 里有没有之前保存过的覆盖值，有就用那个：

   ```c
   // main/provisioning.c，在 Provision_Init() 里
   strlcpy(s_sn, USR_SN, sizeof(s_sn));   // 1. Kconfig 默认值
   // ...同一个函数后面，在 `if (nvs_open(...) == ESP_OK)` 这个块里：
   len = sizeof(s_sn);
   if (nvs_get_str(h, PROV_KEY_SN, tmp, &len) == ESP_OK)
   {
     strlcpy(s_sn, tmp, sizeof(s_sn));   // 2. 如果之前真的保存过，就用 NVS 里的覆盖值
   }
   ```

3. **在 `provisioning.h` 里加一个 getter**——固件其他地方要读到这个已配置的值，唯一的途径就是它：

   ```c
   const char *Provision_GetSN(void);
   ```

   ```c
   // provisioning.c
   const char *Provision_GetSN(void)
   {
     return s_sn;
   }
   ```

4. **在真正用到这个值的地方调用这个 getter**，而不是继续用原来那个 Kconfig 常量——这一步最容易
   被漏掉，因为漏了固件照样能编译、照样能跑，只是会悄悄忽略你这个新指令保存下来的值。`main.c`
   里连接 WiFi 那一行，就是 WiFi SSID/密码这两个字段处理得对的例子：

   ```c
   // main/main.c
   res_val = WiFi_Connect((char *)Provision_GetWifiSsid(), (char *)Provision_GetWifiPassword(), USR_CONCTRY_CODE);
   ```

   （注意第三个参数，WiFi 国家代码，**没有**通过 `Provision_Get*` 这样的访问器——它还是那个原始
   的 Kconfig 常量。目前没有 `SetupWifiCountry` 这个指令；如果你要加一个，这一行调用就正是你
   同时需要改的地方——否则新指令会校验通过、应答成功，但实际上什么效果都没有。）

## 别忘了协议文档

新加一个指令就是改了通信协议——要写进
[protocol/hardware-communication-protocol.md](../protocol/hardware-communication-protocol.md)
的 §1.2，照着现有 `SetupWifi`/`SetupServer`/`SetupDevice` 的表格格式来（请求 JSON 的形状、一个
字段表，以及它跟应答格式的关系）。这份文档也正是
[ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) 用户查"该往手动发送框里
输入什么"时看的那份——具体那一侧是什么样子，见用户手册的
[第 5 章](../manual/05-provision-device-over-serial.zh-CN.md)。

## 检查清单

1. NVS key + 最大长度的缓冲区 + 常驻内存的 static 变量，命名跟现有的对齐（`PROV_KEY_*`、
   `PROV_*_MAX_LEN`、`s_*`）。
2. 一个 `prov_save_*` 函数（打开 NVS、写入、commit、关闭）。
3. 一个 `prov_handle_setup_*` 函数：校验每个字段（必填吗？太长吗？类型对吗？）、保存、更新内存里
   的值、应答。
4. 把指令名字加进 `prov_handle_line()` 的判断链里。
5. 把它的"默认值→NVS 覆盖"加载逻辑加进 `Provision_Init()`。
6. 在 `provisioning.h`/`.c` 里加一个 `Provision_Get*` 访问器。
7. **在真正用到这个值的地方调用这个访问器**——别让第 6 步加的东西没人用。
8. 把新指令写进 `protocol/hardware-communication-protocol.md` 的 §1.2。
9. 构建并烧录到真实硬件上，通过串口发这个指令，确认应答正确、而且这个值真的生效了（这一层没有
   能在主机上跑的测试）。

## 下一步

[固件：新增服务端下发指令](07-firmware-add-server-command.zh-CN.md)。
