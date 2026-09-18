# 固件：接入新传感器

*[English](05-firmware-add-sensor.md)*

给 WS1B 固件加一个传感器涉及三层：**驱动**（跟具体芯片打交道）、**采样函数**（调用驱动，把读数
推进队列）、**payload 构建**（把队列里的数据倒进上报的 JSON——这一层几乎不需要动，因为它本来就是
通用的）。这一章走一遍 `ubibot-open-ws1b` 里已经有的模式，用现有已经接好的几个传感器当模板。

## 1. 字段编号表

`main/MsgType.h` 给每个读数分配一个 1-20 的编号（对应协议 §6 的 `field1`-`field20`）：

```c
// main/MsgType.h
#define TEMP_NUM        1
#define HUMI_NUM        2
#define LIGHT_NUM       3
#define BAT_NUM         4
#define RSSI_NUM        5
#define EXT1_TEMP_NUM   6
#define EXT2_TEMP_NUM   7
```

1-7 号已经被占用了（温度、湿度、环境光、电池电压、WiFi 信号强度，以及两个外接温度探头）。
**8 号是下一个空着的编号**——给你的传感器挑下一个没被用过的编号，最多到 20 号
（`USR_POST_DATA_SUM`，也就是队列容量，本来就是按满量程配的，不需要为多加一个字段去改它）。

## 2. 驱动

每个传感器的底层代码都放在 `app_driver/sensors/` 下（`sht30dis.c`、`ltr308.c`、`stk8323.c`、
`ub_dt_p1.c`），头文件在 `app_driver/<名字>.h`，声明一个 `Init` 和一个读取函数——每次都是同样的
形状：

```c
// app_driver/ltr308.h
extern int LightSensor_Init(void);
extern void LightSensor_value(float *lightvalue);

// app_driver/sht30dis.h
extern void sht30_SingleShotMeasure(float *temp, float *humi);
```

给你的新传感器也照这个样子写驱动：如果芯片需要一次性初始化（配置总线、上电时序），就写一个
`_Init()`；读取函数按它上报几个值就带几个输出指针参数。把新加的 `.c` 文件加进
`main/CMakeLists.txt` 的 `SRCS` 列表——具体长什么样见
《[固件：新增串口配网指令](06-firmware-add-serial-command.zh-CN.md)》（`provisioning.c`/
`command.c` 当初就是这么加进去的）。

## 3. 采样函数

`main.c` 里的 `Sensors_Data_Update()` 会依次调用每个驱动一次，把每个有效读数连同它的字段编号推进
`Data_Queue`：

```c
// main/main.c
void Sensors_Data_Update(void)
{
  float lightvalue=0;
  SensorMessage sMsg={0};
  sMsg.ts = Read_UnixTime();

  LightSensor_value(&lightvalue);  //Read Light sensor
  if(lightvalue!=FAILURE)
  {
    sMsg.sensornum=LIGHT_NUM;  //Message Number
    sMsg.sensorval=lightvalue;  //Message Value
    osi_MsgQWrite(&Data_Queue,&sMsg,OSI_SAVE_WAIT);  //Send Message
  }
  // ...温湿度、电池、两个外接探头也是同样的模式重复一遍
}
```

给你的传感器照这三行来：读一次，跟你的驱动返回的失败标记值比较一下（`ERROR_CODE` 或者
`FAILURE`，看你抄的是哪个现有驱动），只有读数有效的时候才设置 `sensornum`/`sensorval`/`ts` 并
调用 `osi_MsgQWrite`。读取失败就直接不推进队列，不是发一个 0 或者错误值——这一轮上报里就是没有
这个字段。

## 4. payload 构建——通常不用动

`main/json_payload.c` 里的 `Sensors_PostData_Read()` 会把队列里排出来的每条消息，按
`field<sensornum>` 这个 key 塞进上报的 `payloads` 数组：

```c
// main/json_payload.c
snprintf(field,sizeof(field),"field%d",sMsg.sensornum);  //fields number
json_arry = cJSON_CreateObject();
cJSON_AddItemToArray(json_arrys,json_arry);
cJSON_AddNumberToObject(json_arry,"ts",sMsg.ts);
cJSON_AddNumberToObject(json_arry,field,sMsg.sensorval);
```

这段代码对任何字段编号都是通用的——加一个新传感器不需要改它，只有当你要上报的不是普通浮点数这种
新类型的值时才需要动（目前没有这种情况）。

## 5. 顺手在后台给它起个好认的名字（可选）

后端和管理控制台并不知道 `field8` 是什么意思——对它们来说就是一个数字（见
《[架构总览](../architecture/overview.md)》里"没有固定 schema"那条说明）。如果你想让它显示成一个
真正的名字/单位/图标而不是"Field 8"，去后台配一下：可以在图标库（System → Icons，
`GET/POST /api/admin/icons`）里给这款固件的所有设备设个默认值，也可以针对单台设备单独覆盖
（`GET/POST /api/admin/devices/{id}/field-settings`）——见
[api/admin-api.md](../api/admin-api.md)。

## 检查清单

1. 在 `MsgType.h` 里挑下一个空着的字段编号。
2. 在 `app_driver/` 下写驱动（`_Init`/读取函数），加进 `CMakeLists.txt`。
3. 在 `Sensors_Data_Update()` 里加一次调用，照着"读取→校验→入队"的模式来。
4. 构建并烧录到真实设备上（这一层没有能在主机上跑的测试——见
   《[开发环境搭建](01-environment-setup.zh-CN.md)》），确认新字段真的出现在上报里
   （`ubibot-serial-sync` 的日志，或者后台的"监控"页面）。
5. 可以顺手在管理控制台给这个新字段配一个显示名称/图标。

## 下一步

[固件：新增串口配网指令](06-firmware-add-serial-command.zh-CN.md)。
