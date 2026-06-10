# Detecting Dead Sensor Batteries in Home Assistant (Paradox Alarm via PAI)

PAI exposes each alarm zone as a binary sensor (`binary_sensor.zone_front_door_open` etc.) but does not expose individual sensor battery levels for wireless zones — there is no `rf_low_battery` entity in most setups. This solution tracks the last time each zone triggered and notifies you at midnight if any zone hasn't fired in 7+ days.

Two common approaches don't work reliably: `last_changed` resets on HA restart, and the SQL recorder doesn't reliably return pre-reboot data. Instead, this uses a trigger-based template sensor which persists its attributes to disk and survives reboots. An automation updates only the triggered zone's timestamp on each real `on` event — timestamps are never overwritten by a restart.

---

## Setup

### Step 1 — Add the Template Sensor

Add to `configuration.yaml` and restart HA:

```yaml
template:
  - trigger:
      - platform: event
        event_type: zone_last_seen_update
    sensor:
      - name: "Zone Last Seen"
        unique_id: zone_last_seen
        state: "{{ now().strftime('%Y-%m-%d %H:%M') }}"
        attributes:
          data: "{{ trigger.event.data.payload }}"
```

### Step 2 — Create the Automation

Go to **Settings → Automations → + Create Automation → Edit in YAML** and paste the following. Replace the zone entity IDs in both the trigger list and the seed list with your own. Find your exact entity IDs in **Developer Tools → States** by filtering for `binary_sensor.zone_`.

```yaml
alias: Zone Sensor Battery Monitor
description: Seeds on first trigger, updates last seen timestamps on trigger, midnight checks for inactivity
mode: queued
max: 10
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.zone_front_door_open
      - binary_sensor.zone_garage_beam_open
      - binary_sensor.zone_passage_pir_open
      - binary_sensor.zone_dining_room_open
      - binary_sensor.zone_tv_room_open
      - binary_sensor.zone_back_door_beam_open
      # add all your zone sensors here
    to: "on"
  - trigger: time
    at: "00:00:00"
actions:
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ trigger.platform == 'state' }}"
        sequence:
          - event: zone_last_seen_update
            event_data:
              payload: >
                {% set existing = state_attr('sensor.zone_last_seen', 'data') or {} %}
                {% set ts = now().timestamp() | int %}
                {% if existing == {} %}
                  {% set zones = [
                    'binary_sensor.zone_front_door_open',
                    'binary_sensor.zone_garage_beam_open',
                    'binary_sensor.zone_passage_pir_open',
                    'binary_sensor.zone_dining_room_open',
                    'binary_sensor.zone_tv_room_open',
                    'binary_sensor.zone_back_door_beam_open'
                  ] %}
                  {% set ns = namespace(d={}) %}
                  {% for entity_id in zones %}
                    {% set key = entity_id.replace('binary_sensor.zone_', '').replace('_open', '') %}
                    {% set zone_ts = states[entity_id].last_changed.timestamp() | int %}
                    {% set ns.d = dict(ns.d, **{key: zone_ts}) %}
                  {% endfor %}
                  {{ ns.d | to_json }}
                {% else %}
                  {% set key = trigger.entity_id.replace('binary_sensor.zone_', '').replace('_open', '') %}
                  {{ dict(existing, **{key: ts}) | to_json }}
                {% endif %}
      - conditions:
          - condition: template
            value_template: "{{ trigger.platform == 'time' }}"
        sequence:
          - condition: template
            value_template: "{{ state_attr('sensor.zone_last_seen', 'data') not in [none, {}] }}"
          - variables:
              inactive_count: >
                {% set data = state_attr('sensor.zone_last_seen', 'data') %}
                {% set ns = namespace(count=0) %}
                {% for key, ts in data.items() %}
                  {% if (now().timestamp() - ts) > 604800 %}
                    {% set ns.count = ns.count + 1 %}
                  {% endif %}
                {% endfor %}
                {{ ns.count }}
              inactive_list: >
                {% set data = state_attr('sensor.zone_last_seen', 'data') %}
                {% set ns = namespace(result=[]) %}
                {% for key, ts in data.items() %}
                  {% if (now().timestamp() - ts) > 604800 %}
                    {% set name = key.replace('_', ' ').title() %}
                    {% set ns.result = ns.result + [name] %}
                  {% endif %}
                {% endfor %}
                {{ ns.result | join('\n- ') }}
          - condition: template
            value_template: "{{ inactive_count | int > 0 }}"
          - action: persistent_notification.create
            data:
              title: ⚠️ Sensor Battery Check
              message: >
                The following sensors have had no activity for 7+ days —
                batteries may be dead:

                - {{ inactive_list }}
              notification_id: sensor_inactivity_check
```

---

## Notes

- **First run** — on the first sensor trigger, all zones are seeded from `last_changed` as a one-time baseline. After that, only real `on` events update timestamps.
- **Reboot safe** — stored timestamps survive restarts and are never overwritten by a reboot.
- **Threshold** — `604800` is 7 days in seconds. Adjust to suit your setup.
- **Zone list** — entity IDs must appear in both the trigger list and the seed list. PAI names sensors as `binary_sensor.zone_<name>_open`.
- **No HACS required** — built-in HA features only.
- **Overhead** — negligible. Updates fire only on real sensor events.

---

## Example Notification

```
⚠️ Sensor Battery Check

The following sensors have had no activity for 7+ days — batteries may be dead:

- Back Door Beam
- Shed Beam
- Workshop Door
- Flat Bedroom
```

---

## Tested On

- Home Assistant OS 17.3 on Raspberry Pi 4
- Core: 2026.6.1 | Supervisor: 2026.05.1 | Frontend: 20260527.4
- Paradox Alarm Interface (PAI) add-on v3.7.0 (Paradox MG5050 panel)
