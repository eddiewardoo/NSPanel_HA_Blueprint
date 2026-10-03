# NSPanel HA Blueprint: standalone mode (no blueprint automation)

These packages move everything the Home Assistant blueprint configured into your ESPHome YAML.
After flashing, you can delete or disable the blueprint automation for that panel.

Full example (this panel's real config, with every other option commented out):
[`examples/office-nspanel-display-test.yaml`](examples/office-nspanel-display-test.yaml)

## How it works

On boot, the firmware fires an `esphome.nspanel_ha_blueprint` event (`type: boot`). It then waits
until the blueprint has pushed about 50 settings through the `component_text` / `component_val` /
`component_color` actions with `page: mem`: colors, fonts, relay and button config, the version, and so on.

`nspanel_standalone_base.yaml` hooks into that request and applies the same settings locally from
substitutions. The panel finishes booting even when Home Assistant is offline.

The live content the blueprint used to push (chips, values, buttons, weather...) comes from small
feature packages. Each one subscribes directly to the Home Assistant entity it needs (`homeassistant`
platform) and draws on the display itself. Clicks are handled on the device too.

## Setup

1. In your device YAML, add a second remote package next to the core one:

   ```yaml
   packages:
     remote_package:            # core firmware, unchanged
       url: https://github.com/Blackymas/NSPanel_HA_Blueprint
       ref: main
       files: [nspanel_esphome.yaml]
     standalone:
       url: https://github.com/eddiewardoo/NSPanel_HA_Blueprint
       ref: claude/keen-pasteur-hjcvae
       files:
         - standalone/nspanel_standalone_base.yaml          # required
         - path: standalone/nspanel_standalone_home_chip.yaml # one entry per item
           vars: {chip: "01", entity: binary_sensor.front_door, icon: ""}
   ```

2. Put the former blueprint inputs in `substitutions:` (see the list in the example).
3. Buttons that act on Home Assistant use `homeassistant.action`. In Home Assistant, go to
   **Settings → Devices & services → ESPHome → (your panel) → Configure** and enable
   **Allow the device to perform Home Assistant actions**.
4. Flash, then disable the blueprint automation for this panel.

Pin `ref` of **both** packages to the same release when you update. The standalone packages hook
into internal script names of the core firmware.

## Blueprint input → YAML

| Blueprint input | Standalone YAML |
|---|---|
| Global: language, date/time format, colors, decimal separator, temperature unit | substitutions in `nspanel_standalone_base.yaml` (`time_format`, `date_format`, `weekday_names`, ...) |
| `weather_entity` | `nspanel_standalone_home_weather.yaml` (`weather_entity`) |
| `outdoortemp` | `nspanel_standalone_home_outdoor_temp.yaml` (`entity`) |
| `indoortemp` = panel sensor | `embedded_indoor_temperature: "true"` |
| `home_value01..04` | `nspanel_standalone_home_value.yaml` (HA entity) or `..._home_value_local.yaml` (sensor on the panel) |
| `chip01..07` | `nspanel_standalone_home_chip.yaml` |
| `home_button01..07` (custom buttons) | `nspanel_standalone_home_custom_button.yaml` |
| `relay01_icon`, `relay02_icon`, local control / fallback | substitutions `relay01_icon`, `left_button_controls_relay1`, `relay_1_local_fallback`, ... |
| `left/right_button_entity` (HA entity) | `nspanel_standalone_hw_button_ha.yaml` (`side`, `entity`) |
| `left/right_button_entity` = embedded thermostat | `nspanel_standalone_hw_button_climate.yaml` (`side`) |
| `left/right_button_name`, colors, bars | substitutions `left_button_name`, `hw_buttons_bar_color_on`, ... |
| `climate` = embedded thermostat | `is_climate: "true"`, `embedded_climate: "true"` |
| `entity01..32` (+ `_confirm`) | `nspanel_standalone_buttonpage_button.yaml` (`page` 01-04, `button` 01-08, `confirm`) |
| `entities_entity01..32` | `nspanel_standalone_entitypage_entity.yaml` (`page` 01-04, `row` 01-08) |
| QR code | core substitutions `qrcode_initial_value`, `qrcode_title_initial_value` + `qrcode_enabled: "true"` |
| Screensaver | substitutions `screensaver_*` |
| `timezone` | time comes from Home Assistant; optional `time: - id: !extend time_provider` + `timezone:` |

## Icons

Icons are Material Design Icons **code points**, not `mdi:` names, because the name→glyph map
(about 7,000 entries) lives in the blueprint. Look the name up in [`mdi_icons.txt`](mdi_icons.txt)
and write it in **double quotes**: `icon: ""` (mdi:lightbulb-on-outline).

## Not covered in standalone mode

These blueprint features need data or logic that only lives in Home Assistant templates:

- Detail pages opened by a long press: light dimmer and color, cover position, fan, media player, alarm,
  and a Home Assistant climate entity. (The **embedded** thermostat page works: tap the indoor temperature.)
- Weather forecast pages (weather01-05). The home weather icon works.
- Utilities page.
- Brightness % shown on light buttons.
- Translated texts. Set `weekday_names`, `month_names`, `meridiem_*`, `mui_unavailable` and
  `mui_please_confirm` yourself if you want non-English labels.

Notifications still work without the blueprint. Call `esphome.<panel>_notification_show` from any automation.
