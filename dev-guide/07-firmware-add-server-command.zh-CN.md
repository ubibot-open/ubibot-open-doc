# 固件：新增服务端下发指令

*[English](07-firmware-add-server-command.md)*

协议 §9 的指令通道故意做得很窄——管理员可以下发 `reboot` 或 `set_interval`，发了不管，随设备
下一次上报捎带过去——但要加第三个动作，走的是跨三个仓库的固定一条链路。这一章用 `set_interval`
（两个现有动作里更有意思的那个）当参照，把整条链路走一遍。

## 端到端的完整链路

```
后台按钮/表单
        │  POST /api/admin/devices/{id}/commands  {"action":"set_interval","seconds":600}
        ▼
   admin API handler  ──── 校验，然后 ────▶  Device.PendingCmd  （一个列，一台设备，一个队列位）
        │                                                   │
        │                                     设备下一次 POST /api/v1/data/report
        │                                                   ▼
        │                                   Report() 把 PendingCmd 取出来，塞进 "cmd" 字段
        ▼                                                   │
  （在指令送达之前，管理员                                    ▼
   一直能看到 pending_command）              固件的 Command_HandleResponse() 处理它
```

这张图里没有任何应答环节——admin API 那次调用，指令一进队列就算成功了，固件也从来不会回报它到底
有没有真的执行。这是故意的（原因见《[修改通信协议](09-protocol-changes.zh-CN.md)》）；你加的新
动作也要按这个预期去设计。

## 1. 后端：入队和送达

`Device` 上一个列就是整个队列（`server/internal/model/model.go`）：

```go
// PendingCmd is a queued, not-yet-delivered command for this device
// (docs §9), stored as the exact JSON object to embed in the next
// report response's "cmd" field (e.g. {"action":"reboot"}); empty
// means nothing queued. At most one command at a time — setting a new
// one overwrites whatever hadn't been delivered yet. Delivery clears
// it (see store.PopPendingCommand): fire-and-forget, no ack.
PendingCmd string `gorm:"type:text"`
```

`SendDeviceCommand`（`server/internal/api/admin_handlers.go`）是对请求的动作做一个 `switch`——
这正是加第三个动作时该加 `case` 的地方：

```go
var cmdJSON []byte
switch req.Action {
case "reboot":
	cmdJSON, _ = json.Marshal(map[string]any{"action": "reboot"})
case "set_interval":
	if req.Seconds < minReportIntervalSeconds || req.Seconds > maxReportIntervalSeconds {
		adminErr(w, 400, fmt.Sprintf("seconds must be between %d and %d", minReportIntervalSeconds, maxReportIntervalSeconds))
		return
	}
	cmdJSON, _ = json.Marshal(map[string]any{"action": "set_interval", "seconds": req.Seconds})
default:
	adminErr(w, 400, "unsupported action")
	return
}
// ...s.Store.SetPendingCommand(uint(id), string(cmdJSON))
```

注意 `set_interval` 把 `seconds` 限制在 60-86400 这个范围**是在后端做的**——代码注释里写清楚了
为什么："固件本身不会做这个范围校验——给它什么 seconds 值它就用什么——所以在指令入队之前，把值
校验清楚是 admin API 的责任。"任何带数值参数、或者参数取值没有天然上限的新动作，都应该出于同样
的理由在这里做校验：假定固件收到什么就会照做，不会反过来校验。

送达这一步是通用的，加新动作不需要改它——`Report()`（`server/internal/api/device_handlers.go`）
只是把队列里排出来的东西原样塞进去：

```go
resp := protocol.ReportResponse{C: protocol.CodeOK, T: now.Unix()}
if cmdJSON, err := s.Store.PopPendingCommand(dev.ID); err != nil {
	log.Printf("pop pending command for device %d: %v", dev.ID, err)
} else if cmdJSON != "" {
	resp.Cmd = json.RawMessage(cmdJSON)
}
```

## 2. 固件：执行它

`main/command.c` 里的 `Command_HandleResponse()` 是后端那个 `switch` 的镜像——对 `cmd.action` 做
一串 `if`/`else if`：

```c
// main/command.c
if (strcmp(action->valuestring, "reboot") == 0)
{
  esp_restart();  // never returns
}
else if (strcmp(action->valuestring, "set_interval") == 0)
{
  cJSON *seconds = cJSON_GetObjectItem(cmd, "seconds");
  if (!cJSON_IsNumber(seconds) || (seconds->valuedouble <= 0))
  {
    ESP_LOGW(TAG, "set_interval command missing a positive \"seconds\"");
    return;
  }
  command_save_report_interval((uint32_t)seconds->valuedouble);
}
else
{
  ESP_LOGW(TAG, "unrecognized cmd.action: %s", action->valuestring);
}
```

新动作在这里加自己的 `else if` 分支：如果需要持久化什么东西，就照着 `set_interval` 的样子来
（打开 NVS、写入、commit、关闭，然后更新内存里的值——跟《[固件：新增串口配网指令](06-firmware-add-serial-command.zh-CN.md)》
里 `provisioning.c` 那套完全一样的模式）；如果不需要持久化，就照着 `reboot` 的样子，直接执行。

两个接线的地方，因为各自只有一处，很容易做对：

- **在哪里被调用**：`Command_HandleResponse()` 是从 `app_driver/net/http_client.c` 的
  `Parse_Response()` 里被调用的，每一个解析出来的 HTTP 响应都会走一遍——没有 `cmd` 字段的时候
  （比如时间同步的响应）它什么都不做，所以加新动作不需要在这里多做什么。
- **持久化的值在哪里被读回来用**：如果你的动作会改变某个持续生效的行为（就像 `set_interval`
  那样），就在 `command.h`/`.c` 里加一个 `Command_Get*` 访问器，然后在那个行为真正发生的地方
  调用它——`main.c` 里的休眠调用就是现有的例子：

  ```c
  Enter_Sleep(Command_GetReportIntervalSeconds());  // interval is the server-commanded value if one was ever set, else DEFAULT_FN
  ```

## 3. 测试

`command_handlers_test.go` 用两端真实的 HTTP 接口把整条链路验证了一遍——通过 admin API 入队，
确认它显示成 `pending_command`，像设备一样发一次上报，确认响应里带上了它，再确认下一次上报里
它已经没了：

```go
// server/internal/api/command_handlers_test.go
func TestSendDeviceCommand_RebootDeliveredOnceThenCleared(t *testing.T) {
	env := newTestEnv(t)
	adminAuth := env.createSuperAdmin(t, "admin", "s3cret-pw")

	rec, body := env.do(t, "POST", fmt.Sprintf("/api/admin/devices/%d/commands", env.dev.ID),
		map[string]any{"action": "reboot"}, adminAuth)
	// ...断言 rec.Code == 200

	// Visible as pending before it's ever delivered.
	rec, body = env.do(t, "GET", fmt.Sprintf("/api/admin/devices/%d", env.dev.ID), nil, adminAuth)
	// ...断言 dev["pending_command"] != nil

	// First report after queuing: the response carries the command.
	rec, body = env.do(t, "POST", "/api/v1/data/report",
		report(testSN, env.now.Unix(), map[string]any{"field1": 20}), nil)
	// ...断言 body["cmd"] == {"action":"reboot"}
}
```

固件那一侧没有对应的测试（`ubibot-open-ws1b` 没有能在主机上跑的测试套件，见
《[开发环境搭建](01-environment-setup.zh-CN.md)》）——验证新动作的方式是构建、烧录到真实硬件上
（如果手头没有设备，就仔细读一遍代码路径——模拟器没有实现指令通道，因为这是一个后台触发的功能，
纯设备模拟器本身不需要跑这一段），从管理控制台下发指令，然后在设备下一次上报之后看串口日志。

## 检查清单

1. 在 `SendDeviceCommand` 的 `switch` 里加新动作（后端），任何参数都要给一个合理的边界校验——
   固件不会帮你校验。
2. 在 `Command_HandleResponse` 的 `if`/`else if` 链里加同一个动作（固件）。
3. 如果它会改变某个持续生效的行为，加一个 `Command_Get*` 访问器，在那个行为真正发生的地方调用它。
4. 照着 `TestSendDeviceCommand_RebootDeliveredOnceThenCleared` 的样子加一个测试。
5. 把新动作写进 `protocol/hardware-communication-protocol.md` 的 §9。
6. 如果管理控制台需要一个新的表单字段来触发它，UI 那一侧的写法见
   《[新增后台页面](04-admin-add-page.zh-CN.md)》。

## 下一步

[用模拟器迭代后端](08-testing-with-the-simulator.zh-CN.md)。
