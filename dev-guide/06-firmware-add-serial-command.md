# Firmware: Add a Serial Provisioning Command

*[中文](06-firmware-add-serial-command.zh-CN.md)*

`main/provisioning.c` implements the three commands protocol §1.2 defines today —
`SetupWifi`, `SetupServer`, `SetupDevice` — and a new one follows exactly the same shape. This
chapter walks through `SetupDevice`, the simplest of the three (a single string field), as the
template.

## The shape every command follows

Each command is a small cluster of pieces, all in `provisioning.c`:

```c
// main/provisioning.c
#define PROV_KEY_SN "sn"                    // 1. an NVS key
#define PROV_SN_MAX_LEN 65                  // 2. a max length for its buffer(s)
static char s_sn[PROV_SN_MAX_LEN];          // 3. RAM-resident active value

// 4. save it to NVS
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

// 5. validate, save, update RAM, ack
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

Every field gets exactly this treatment: required fields rejected with `c:1` and a specific
message if missing/empty/too long; a save failure reported as `c:2`; success updates both NVS
*and* the RAM copy (so the change also applies to the device's current run, not just its next
boot) and acks with `c:0`.

## Wiring a new command in

1. **Dispatch it** — `prov_handle_line()` is a plain `strcmp` chain on the `command` field; add
   your command's name to it:

   ```c
   // main/provisioning.c, inside prov_handle_line()
   else if (strcmp(command->valuestring, "SetupDevice") == 0)
   {
     prov_handle_setup_device(root);
   }
   // your new command's else-if goes here, same shape
   ```

2. **Load its default and NVS override in `Provision_Init()`** — every field follows the same
   two-step: start from a Kconfig compile-time default, then check NVS for a previously-saved
   override and use that instead if present:

   ```c
   // main/provisioning.c, inside Provision_Init()
   strlcpy(s_sn, USR_SN, sizeof(s_sn));   // 1. Kconfig default
   // ...later in the same function, inside the `if (nvs_open(...) == ESP_OK)` block:
   len = sizeof(s_sn);
   if (nvs_get_str(h, PROV_KEY_SN, tmp, &len) == ESP_OK)
   {
     strlcpy(s_sn, tmp, sizeof(s_sn));   // 2. NVS override, if one was ever saved
   }
   ```

3. **Expose a getter in `provisioning.h`** — the only way the rest of the firmware reads a
   provisioned value:

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

4. **Call the getter from wherever the value is actually used**, instead of the raw Kconfig
   constant — this is the step that's easy to forget, since the firmware will still compile and
   run without it, just silently ignoring every value your new command saves. `main.c`'s WiFi
   connect call is a good example of getting this right for the two WiFi fields:

   ```c
   // main/main.c
   res_val = WiFi_Connect((char *)Provision_GetWifiSsid(), (char *)Provision_GetWifiPassword(), USR_CONCTRY_CODE);
   ```

   (Notice the third argument, the WiFi country code, is *not* wired through a
   `Provision_Get*` accessor — it's still the raw Kconfig constant. There is no
   `SetupWifiCountry` command today; if you added one, this is exactly the call site you'd need
   to update too, or the new command would validate and ack successfully while having no actual
   effect.)

## Don't forget the protocol doc

A new command is a wire-format change — document it in
[protocol/hardware-communication-protocol.md](../protocol/hardware-communication-protocol.md)'s
§1.2, following the existing table format for `SetupWifi`/`SetupServer`/`SetupDevice` (request
JSON shape, a field table, and how it interacts with the ack format). This is the same doc
[ubibot-serial-sync](https://github.com/ubibot-open/ubibot-serial-sync) users read to know what to
type into the manual-send box — see [chapter 5](../manual/05-provision-device-over-serial.md) of
the user manual for what that looks like from that side.

## Checklist

1. NVS key + max-length buffer + RAM-resident static, following the existing naming (`PROV_KEY_*`,
   `PROV_*_MAX_LEN`, `s_*`).
2. A `prov_save_*` function (open NVS, write, commit, close).
3. A `prov_handle_setup_*` function: validate each field (required? too long? right type?), save,
   update the RAM copy, ack.
4. Add the command name to `prov_handle_line()`'s dispatch chain.
5. Add its default-then-NVS-override loading to `Provision_Init()`.
6. Add a `Provision_Get*` accessor to `provisioning.h`/`.c`.
7. **Call that accessor from wherever the value is actually consumed** — don't leave step 6
   unused.
8. Document the new command in `protocol/hardware-communication-protocol.md` §1.2.
9. Build and flash to real hardware, send the command over serial, confirm the ack and that the
   value actually takes effect (there's no host-run test for this layer).

## Next

[Firmware: Add a Server-Issued Command](07-firmware-add-server-command.md).
