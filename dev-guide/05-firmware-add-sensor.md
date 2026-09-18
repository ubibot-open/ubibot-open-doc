# Firmware: Wire In a New Sensor

*[中文](05-firmware-add-sensor.zh-CN.md)*

Adding a sensor to the WS1B firmware touches three layers: the **driver** (talks to the actual
chip), the **sampling function** (calls the driver, pushes readings onto a queue), and the
**payload builder** (drains that queue into the report's JSON — this one you almost never need to
touch, since it's already generic). This chapter walks through the existing pattern in
`ubibot-open-ws1b`, using the already-wired sensors as the template.

## 1. The field number registry

`main/MsgType.h` assigns each reading a number, 1-20 (protocol §6's `field1`-`field20`):

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

Numbers 1-7 are taken (temperature, humidity, ambient light, battery voltage, WiFi signal
strength, and two external temperature probes). **8 is the next free number** — pick the next
unused one for your sensor, up to 20 (`USR_POST_DATA_SUM`, the queue's capacity, is already sized
for the full range — no need to touch it for one more field).

## 2. The driver

Each sensor's low-level code lives under `app_driver/sensors/` (`sht30dis.c`, `ltr308.c`,
`stk8323.c`, `ub_dt_p1.c`), with its header at `app_driver/<name>.h` declaring an `Init` and a
`read` function — the same shape every time:

```c
// app_driver/ltr308.h
extern int LightSensor_Init(void);
extern void LightSensor_value(float *lightvalue);

// app_driver/sht30dis.h
extern void sht30_SingleShotMeasure(float *temp, float *humi);
```

Write your new sensor's driver the same way: an `_Init()` if the chip needs one-time setup (bus
configuration, a power-on sequence), and a read function taking an output pointer per value it
reports. Add the new `.c` file to `main/CMakeLists.txt`'s `SRCS` list — see
[Firmware: Add a Serial Provisioning Command](06-firmware-add-serial-command.md) for what that
looks like (`provisioning.c`/`command.c` were added the same way).

## 3. The sampling function

`Sensors_Data_Update()` in `main.c` calls every driver once, and pushes each valid reading onto
`Data_Queue` with its field number:

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
  // ...same pattern repeated for temp/humidity, battery, and the two external probes
}
```

Follow the same three lines for your sensor: read it, check the result against whatever failure
sentinel your driver returns (`ERROR_CODE` or `FAILURE`, depending on which existing driver you
copied), and — only if the reading is valid — set `sensornum`/`sensorval`/`ts` and
`osi_MsgQWrite`. A failed reading is simply not enqueued, not sent as a zero or an error value; the
report just won't include that field for this cycle.

## 4. The payload builder — usually untouched

`Sensors_PostData_Read()` in `main/json_payload.c` drains every queued message into the report's
`payloads` array, keyed by `field<sensornum>`:

```c
// main/json_payload.c
snprintf(field,sizeof(field),"field%d",sMsg.sensornum);  //fields number
json_arry = cJSON_CreateObject();
cJSON_AddItemToArray(json_arrys,json_arry);
cJSON_AddNumberToObject(json_arry,"ts",sMsg.ts);
cJSON_AddNumberToObject(json_arry,field,sMsg.sensorval);
```

This is already generic across every field number — you don't need to change it for a new sensor,
only for a new *kind* of value that isn't a plain float (there is no such case today).

## 5. Give it a friendly label in the admin console (optional)

The backend and admin console have no idea what `field8` means — it's just a number (see the
[Architecture Overview](../architecture/overview.md)'s "no fixed schema" note). If you want it to
show up with a real name/unit/icon instead of "Field 8", set that up on the admin side: either a
default in the icon library (System → Icons, `GET/POST /api/admin/icons`) for every device of this
firmware, or a per-device override (`GET/POST /api/admin/devices/{id}/field-settings`) — see
[api/admin-api.md](../api/admin-api.md).

## Checklist

1. Pick the next free field number in `MsgType.h`.
2. Write the driver (`_Init`/read function) under `app_driver/`, add it to `CMakeLists.txt`.
3. Add a call to it in `Sensors_Data_Update()`, following the read → validate → enqueue pattern.
4. Build and flash to a real device (there's no host-run test for this layer — see
   [Environment Setup](01-environment-setup.md)) and confirm the new field shows up in a report
   (`ubibot-serial-sync`'s log, or the backend's `Monitor` page).
5. Optionally, set a display label/icon for the new field in the admin console.

## Next

[Firmware: Add a Serial Provisioning Command](06-firmware-add-serial-command.md).
