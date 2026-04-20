# Energy Monitoring Setup

## What Was Added

### 1. Template Sensors (Power - W)

#### `sensor.home_total_consumption_power`
- **Purpose**: Total home consumption (all loads)
- **Calculation**: power_2 (house) + power_3 (solar pump) + power_4 (well pump)
- **Unit**: W
- **Location**: configuration.yml:163-172

#### `sensor.solar_curtailment_power`
- **Purpose**: Solar power being wasted when battery is full
- **Condition**: Active when:
  - Charging status = 'Standby'
  - Charging current > 0.1A
  - Charging power > 5W
  - Battery SOC ≥ 99%
- **Unit**: W
- **Location**: configuration.yml:174-186

### 2. Integration Sensors (Energy - kWh)

These convert power (W) to energy (kWh) using the `integration` platform (Riemann sum):

#### `sensor.home_total_consumption_energy`
- **Source**: sensor.home_total_consumption_power
- **Unit**: kWh
- **Method**: left

#### `sensor.solar_curtailment_energy`
- **Source**: sensor.solar_curtailment_power
- **Unit**: kWh
- **Method**: left

#### `sensor.grid_charge_energy`
- **Source**: sensor.sonoff_ab300012ee_power_1
- **Unit**: kWh
- **Method**: left

## How to Apply Changes

### Step 1: Validate Configuration
```bash
# SSH into Home Assistant
ha core check
```

### Step 2: Restart Home Assistant
**Option A: Full Restart (Recommended)**
- Settings → System → Restart

**Option B: Reload YAML (Faster)**
- Developer Tools → YAML → Click:
  - "Template Entities"
  - "All YAML Configuration"

### Step 3: Verify Sensors
Go to Developer Tools → States and search for:
- `sensor.home_total_consumption_power`
- `sensor.solar_curtailment_power`
- `sensor.home_total_consumption_energy`
- `sensor.solar_curtailment_energy`
- `sensor.grid_charge_energy`

## Energy Dashboard Configuration

### Step 1: Add Solar Production
1. Settings → Dashboards → Energy
2. Solar Panels → Add Solar Production
3. Select: `sensor.mppt_solar_controller_energy_generated_today`
4. **Note**: If this sensor is in Wh, you may need to convert it to kWh first

### Step 2: Add Grid Consumption
1. Grid Consumption → Add
2. Select: `sensor.grid_charge_energy`

### Step 3: Add Home Consumption
1. Home Energy Monitor → Add
2. Select: `sensor.home_total_consumption_energy`

### Step 4: Optional - Track Curtailment Loss
1. Individual Devices → Add
2. Select: `sensor.solar_curtailment_energy`
3. Name it "Solar Waste" or "Curtailment"

## Important Notes

### Energy Units
- **Power sensors** (W) are instantaneous measurements
- **Energy sensors** (kWh) accumulate over time
- Home Assistant Energy Dashboard requires **kWh** units
- `unit_prefix: k` converts Wh → kWh automatically

### MPPT Energy Sensor
If `sensor.mppt_solar_controller_energy_generated_today` is in **Wh** (not kWh), you need to add a conversion:

```yaml
template:
  - sensor:
      - name: "Solar Production Today kWh"
        unique_id: solar_production_kwh
        unit_of_measurement: "kWh"
        device_class: energy
        state_class: total_increasing
        state: >
          {{ (states('sensor.mppt_solar_controller_energy_generated_today') | float(0) / 1000) | round(2) }}
```

Then use `sensor.solar_production_today_kwh` in Energy Dashboard instead.

### Integration Sensor Behavior
- Starts at 0 after each HA restart
- Accumulates energy over time
- Resets automatically at midnight (if configured in Energy Dashboard)
- Uses "left" method = sampling at interval start

## Troubleshooting

### Sensors showing "unavailable"
- Check that source sensors exist and have valid values
- Verify entity IDs are correct
- Check logs: Settings → System → Logs

### Energy Dashboard not showing data
- Energy sensors need `state_class: total_increasing` or use integration platform
- Ensure sensors are in kWh (not Wh)
- Wait 1-2 hours for data to accumulate
- Check: Developer Tools → Statistics

### Negative energy values
- Change integration method from "left" to "trapezoidal"
- Ensure source power sensors never go negative

## Energy Flow Summary

```
Solar Panels (2×450W)
        ↓
   MPPT Controller → sensor.mppt_solar_controller_charging_power (W)
        ↓
   Battery 24V 100Ah
        ↓
   Inverter 2-4kW
        ↓
   ┌────────────────┬──────────────┬─────────────┐
   │                │              │             │
House Load    Solar Pump    Well Pump    Grid Charge
power_2         power_3       power_4      power_1
   │                │              │             │
   └────────────────┴──────────────┴─────────────┘
                      ↓
          Home Total Consumption
```

## Files Modified
- `configuration.yml` - Added template sensors and integration platform
