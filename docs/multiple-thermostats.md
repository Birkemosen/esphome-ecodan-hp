# Multiple Thermostats (e.g. Zigbee) with Auto-Adaptive Control

This guide describes how to use many room thermostats (e.g. 12 Zigbee2MQTT UFH zone controllers) with the Ecodan ESPHome controller and Auto-Adaptive Control. The heat pump only has **two** room-temperature inputs (`temperature_feedback_z1` and `temperature_feedback_z2`). You aggregate your thermostats in Home Assistant into one value per heat-pump zone and push that to the ESPHome device. No firmware changes are required.

---

## Typical Setup: 12 UFH Zones with Smart Zone Controllers

A common setup is 12 underfloor heating (UFH) zones, each with its own **smart controller** (e.g. in Zigbee2MQTT) that:

- Balances flow per room and ensures equal heat distribution between rooms
- Measures **return temperature** per room
- Enforces a minimum flow per room (e.g. 25% open)

In this architecture the **Ecodan is the primary heat source**: it sets the flow temperature for the whole system. The **zone controllers handle distribution**: they take that primary flow and balance it across the 12 rooms. The ESPHome/Ecodan controller therefore only needs **one or two aggregate values** (per heat-pump zone) to drive Auto-Adaptive—"how much demand is there?"—so it can set the right primary flow temperature.

---

## Step 1: Map Rooms to Zone 1 and Zone 2

Decide which rooms belong to **Zone 1** (e.g. downstairs or circuit 1) and which to **Zone 2** (e.g. upstairs or circuit 2), based on how your heat pump circuits are wired.

- **Single-zone heat pump:** Put all 12 rooms into one logical zone and only use `temperature_feedback_z1` (leave Z2 unused or duplicate).
- **Two-zone heat pump:** For example 6 rooms per zone by floor or by manifold.

---

## Step 2: Choose an Aggregation Strategy

For each heat-pump zone you need **one** representative value from the rooms in that zone.

### Same setpoint in all rooms

If every room has the same target (e.g. 21°C), you can aggregate by **raw temperature**:

- **Minimum room temp (coldest room)** – heating: heat pump drives flow until the coldest room reaches target; zone controllers handle distribution and minimum flow.
- **Minimum return temp (coldest circuit)** – heating: reflects highest demand; good load indicator for UFH (use return temp entities from your smart controllers if exposed in Zigbee2MQTT).
- **Average room or return** – balanced; heat pump follows overall load.
- **Maximum (warmest)** – useful for cooling to avoid overcooling.

### Different setpoints per room

If rooms have **different setpoints** (e.g. living 21°C, bedroom 18°C, bathroom 23°C), aggregating by raw temperature is misleading: the "coldest room" by value might already be at its setpoint (e.g. bedroom 18°C with 18°C target), while another room is still below target (e.g. living 20°C with 21°C target). The controller should respond to **unmet demand**, not raw cold.

**Aggregate by demand (error from setpoint):**

- For each room, heating **error** = `setpoint_i - current_temp_i` (how far below setpoint).
- The room with the **largest positive error** has the most unmet demand.
- Feed the Ecodan an **effective room temperature** so the algorithm sees that worst error:
  - **effective_room_temp** = `zone_setpoint - max(0, max(setpoint_i - current_temp_i))` over all rooms in that zone.
  - Then the algorithm's error = `zone_setpoint - effective_room_temp` = worst room's error.
- **zone_setpoint** = the single target you use on the heat pump for that zone. A good choice is **max of all room setpoints** (e.g. 23°C if bathroom is 23°C). Set the heat pump Zone 1/2 setpoint to this same value. When every room is at or above its setpoint, the controller correctly backs off.

**Cooling:** Use **error_i = current_temp_i - setpoint_i** (room above setpoint); effective_room_temp = zone_setpoint + max(current_temp_i - setpoint_i) so the algorithm sees the warmest-over-target room.

---

## Step 3: Implement in Home Assistant

### 3.1 Set Room Temperature source on the Ecodan

In Home Assistant, set the Ecodan entity **"Auto-Adaptive: Room Temperature source"** to **"Home Assistant / REST API"**. The controller will then use the values you push to `temperature_feedback_z1` and `temperature_feedback_z2` instead of the heat pump's built-in thermostat.

### 3.2 Create template sensors

Create one template sensor per heat-pump zone that computes the aggregate (representative temp or effective room temp). Replace entity IDs with your Zigbee2MQTT entities.

#### Example: Same setpoint – minimum room temperature (Zone 1)

```yaml
# configuration.yaml or template.yaml
template:
  - sensor:
      - name: "Zone 1 representative temperature"
        unique_id: z1_rep_temp
        unit_of_measurement: "°C"
        state_class: measurement
        device_class: temperature
        state: >-
          {% set rooms = [
            states('sensor.room1_temperature'),
            states('sensor.room2_temperature'),
            states('sensor.room3_temperature'),
            states('sensor.room4_temperature'),
            states('sensor.room5_temperature'),
            states('sensor.room6_temperature')
          ] %}
          {{ rooms | select('defined') | map('float', 0) | reject('eq', 0) | min | round(1) }}
```

Repeat for Zone 2 with the other room (or return) temperature entities if you use two zones. Use `min` for "coldest room", `max` for "warmest", or a custom average.

#### Example: Different setpoints – effective room temperature (Zone 1)

You need both **current temperature** and **setpoint** per room from Zigbee2MQTT. Set `zone_setpoint` to the value you use as the heat pump Zone 1 setpoint (e.g. max of all room setpoints).

```yaml
template:
  - sensor:
      - name: "Zone 1 effective room temperature"
        unique_id: z1_effective_room_temp
        unit_of_measurement: "°C"
        state_class: measurement
        device_class: temperature
        state: >-
          {% set zone_setpoint = 23.0 %}
          {% set errors = [
            (states('number.room1_setpoint') | float(20)) - (states('sensor.room1_temperature') | float(0)),
            (states('number.room2_setpoint') | float(21)) - (states('sensor.room2_temperature') | float(0)),
            (states('number.room3_setpoint') | float(18)) - (states('sensor.room3_temperature') | float(0)),
            (states('number.room4_setpoint') | float(21)) - (states('sensor.room4_temperature') | float(0)),
            (states('number.room5_setpoint') | float(22)) - (states('sensor.room5_temperature') | float(0)),
            (states('number.room6_setpoint') | float(23)) - (states('sensor.room6_temperature') | float(0))
          ] %}
          {% set max_err = [errors | select('defined') | map('float', 0) | max | default(0), 0] | max %}
          {{ (zone_setpoint - max_err) | round(1) }}
```

Use your actual Zigbee entity IDs for setpoints (e.g. `climate.room1` attribute `temperature` for setpoint, or a `number.room1_setpoint`). Repeat for Zone 2 with the other rooms if you use two zones.

### 3.3 Automations that push the aggregate to the heat pump

When the template sensor(s) change, call `number.set_value` for the Ecodan feedback number(s). Replace the sensor name with the one you created (representative or effective room temperature).

**Zone 1:**

```yaml
- id: SyncZ1AggregateToEcodan
  alias: Sync Zone 1 (12 rooms) temp to Ecodan
  trigger:
    - platform: state
      entity_id:
        - sensor.zone_1_representative_temperature
        # or sensor.zone_1_effective_room_temperature
  action:
    - service: number.set_value
      target:
        entity_id: number.ecodan_heatpump_auto_adaptive_current_room_temperature_feedback_z1
      data:
        value: "{{ states('sensor.zone_1_representative_temperature') | float(0) }}"
```

**Zone 2** (if you use two zones): duplicate the automation and use `sensor.zone_2_...` and `number.ecodan_heatpump_auto_adaptive_current_room_temperature_feedback_z2`. Adjust the entity IDs to match your ESPHome device name (e.g. `number.ecodan_heatpump_...` may be `number.<your_device_name>_auto_adaptive_current_room_temperature_feedback_z1`).

### 3.4 Setpoints (target temp) on the heat pump

You have one Zone 1 and one Zone 2 setpoint on the heat pump.

- **Same setpoint in all rooms:** Set the heat pump Zone 1/2 setpoint to that value (e.g. 21°C). You can sync from a "master" Zigbee thermostat or leave it manual.
- **Different setpoints per room:** Set the heat pump Zone 1/2 setpoint to **zone_setpoint** (e.g. 23°C = max of room setpoints), matching the value used in the effective-room-temperature template. Then when all rooms are satisfied, the controller correctly backs off.

---

## Summary

| Item | Action |
|------|--------|
| **Room Temperature source** | Set to "Home Assistant / REST API" on the Ecodan. |
| **Mapping** | Assign rooms to Zone 1 / Zone 2 by your heating layout. |
| **Same setpoint** | Use a representative temp (e.g. min room or min return). |
| **Different setpoints** | Use effective room temp: `zone_setpoint - max(0, max(setpoint_i - current_temp_i))`; set heat pump setpoint to `zone_setpoint`. |
| **Automation** | On template sensor state change, call `number.set_value` for `temperature_feedback_z1` (and `_z2`). |
| **Firmware** | No changes; all aggregation in Home Assistant. |

For more on Auto-Adaptive Control, see [auto-adaptive.md](auto-adaptive.md).
