# OpenTherm Master for ESPHome

An external ESPHome component to control a boiler (or other supported HVAC appliances) over the OpenTherm protocol. Note that special hardware is required, like the [DIYLESS Master OpenTherm Shield](https://diyless.com/product/master-opentherm-shield) or [Ihor Melnyk's OpenTherm Adapter](http://ihormelnyk.com/opentherm_adapter). This component acts only as an OpenTherm master (i.e. a thermostat or controller) and not as a slave or gateway. You can no longer use your existing thermostat if you control your boiler through ESPHome with this component.

We aim for maximum flexibility in this component by exposing most of the information available through the OpenTherm protocol, while allowing all configuration in YAML. (No custom component code required!) Since every boiler and every situation is different, you have to play around a bit with the sensors you'd want to read. There is no requirement for a boiler to support everything in the protocol, so not every sensor in this component will work with your boiler. (For example, my Remeha Avanta does not report `ch_pressure`, `dhw_flow_rate` or `t_dhw`.) <!-- #2: We try to be smart about this and not send request messages for these if the boiler consistently indicates it doesn't understand the message or the data is unavailable. You'll find warning messages indicating this behaviour in the ESPHome logs. -->

This component uses [@ihormelnyk's OpenTherm Library](https://github.com/ihormelnyk/opentherm_library) (MIT licensed) as its communication layer. The message loop is inspired by code for the [DIYLESS ESP32 Wi-Fi Thermostat](https://github.com/diyless/esp32-wifi-thermostat) (MIT licensed).

Alternatives:
- [ESPHome-OpenTherm by @rsciriano](https://github.com/rsciriano/ESPHome-OpenTherm), a custom component based on the same library as this project
- [esphome-opentherm by @wichers](https://github.com/wichers/esphome-opentherm), which works as a gateway, rather than a master, allowing you to combine the system with your own thermostat
- And more options if you [search on GitHub](https://github.com/search?q=esphome+opentherm)

## Quick glossary

- CH: Central Heating
- DHW: Domestic Hot Water

## Usage

The OpenTherm Master component is available as an external component in ESPHome and can be included in your configuration as follows:

```yaml
external_components:
  source: github://ananyevgv/esphome-opentherm_custom_component
  refresh: 0s

```

This references the main branch, which is cool if you want to stay up to date, but may also break your configuration if breaking changes happen here. A better idea would be to reference a specific version, see the tags for available versions. Instead of a specific version, you could also choose to follow a major version by specifying `@v1` etc.

Then you can define the OpenTherm hub in your configuration:

```yaml
opentherm:
  in_pin: 4
  out_pin: 5
  master_id: 4 # default 0, baxi 4, 211
```

### Usage as a thermostat

The most important function for a thermostat is to set the boiler temperature setpoint. This component has three ways to provide this input: using a sensor from which the setpoint can be read, using a [number](https://esphome.io/components/number/index.html), or defining an output to which other components can write. For most users, the last option is the most useful one, as it can be combined with the [PID Climate](https://esphome.io/components/climate/pid.html) component to create a thermostat that works as you would expect a thermostat to work. See [thermostat-pid-basic.yaml](examples/thermostat-pid-basic.yaml) for an example.

### Numerical input

There are three ways to set an input value:

- As an input sensor, defined in the hub configuration:
  
  ```yaml
  opentherm:
    t_set: setpoint_sensor

  sensor:
    - platform: homeassistant
      id: setpoint_sensor
      entity_id: sensor.boiler_setpoint
  ```

  This can be useful if you have an external thermostat-like device that provides the setpoint as a sensor.
- As a number:
  
  ```yaml
  number:
    - platform: opentherm
      t_set:
        name: Boiler Setpoint
  ```

  This is useful if you want full control over your boiler and want to manually set all values.
- As an output:
  
  ```yaml
  output:
    - platform: opentherm
      t_set:
        id: setpoint
  ```
  
  This is especially useful in combination with the PID Climate component:

  ```yaml
  climate:
    - platform: pid
      heat_output: setpoint
      # ...
  ```

For the output and number variants, there are four more properties you can configure beyond those included in the output and number components by default:

- `min_value` (float): The minimum value. For a number this is the minimum value you are allowed to input. For an output this is the number that will be sent to the boiler when the output is at 0%.
- `max_value` (float): The maximum value. For a number this is the maximum value you are allowed to input. For an output this is the number that will be sent to the boiler when the output is at 100%.
- `auto_max_value` (boolean): Automatically configure the maximum value to a value reported by the boiler. Not available for all inputs.
- `auto_min_value` (boolean): Automatically configure the minimum value to a value reported by the boiler. Not available for all inputs.

The following inputs are available:

<!-- BEGIN schema_docs:input -->
- `t_set`: Control setpoint: temperature setpoint for the boiler's supply water (°C)  
  Default `min_value`: 0  
  Default `max_value`: 100  
  Supports `auto_max_value`
- `t_set_ch2`: Control setpoint 2: temperature setpoint for the boiler's supply water on the second heating circuit (°C)  
  Default `min_value`: 0  
  Default `max_value`: 100  
  Supports `auto_max_value`
- `cooling_control`: Cooling control signal (%)  
  Default `min_value`: 0  
  Default `max_value`: 100
- `t_dhw_set`: Domestic hot water temperature setpoint (°C)  
  Default `min_value`: 0  
  Default `max_value`: 127  
  Supports `auto_min_value`  
  Supports `auto_max_value`
- `max_t_set`: Maximum allowable CH water setpoint (°C)  
  Default `min_value`: 0  
  Default `max_value`: 127  
  Supports `auto_min_value`  
  Supports `auto_max_value`
- `t_room_set`: Current room temperature setpoint (informational) (°C)  
  Default `min_value`: -40  
  Default `max_value`: 127
- `t_room_set_ch2`: Current room temperature setpoint on CH2 (informational) (°C)  
  Default `min_value`: -40  
  Default `max_value`: 127
- `t_room`: Current sensed room temperature (informational) (°C)  
  Default `min_value`: -40  
  Default `max_value`: 127
<!-- END schema_docs:input -->

### Switch

For five status codes, switches are available to toggle them manually. The same values can be set in the hub configuration, like so:

```yaml
- platform: opentherm
  ch_enable: 
  dhw_enable: 
  cooling_enable:
  otc_active:
  ch2_active:
  lock_out_reset:
```

This can be used to set the value without the need for a switch if you'd never want to toggle it after the initial configuration. The default values for these configuration options are listed below.

For enabling of central heating and cooling, the enable-flag is only sent to the boiler if the following conditions are met:
- the flag is set to true in the hub configuration,
- the switch is on, if it is configured,
- the setpoint or cooling control value is not 0, if it is configured.

For domestic hot water and outside temperature compensation, only the first two conditions are necessary.

The last point ensures that central heating is not enabled if no heating is requested as indicated by a setpoint of 0. If you use a number as the setpoint input and use a minimum value higher than 0, you NEED to use the ch_enable switch to turn off your central heating. In that case the flag will be set to true in the hub configuration, and setpoint is always larger than 0, so including a switch is the only way you can turn off central heating. (This also holds for cooling and CH2.)

The following switches are available:

<!-- BEGIN schema_docs:switch -->
- `ch_enable`: Central Heating enabled  
  Defaults to *True*
- `dhw_enable`: Domestic Hot Water enabled  
  Defaults to *True*
- `cooling_enable`: Cooling enabled  
  Defaults to *False*
- `otc_active`: Outside temperature compensation active  
  Defaults to *False*
- `ch2_active`: Central Heating 2 active  
  Defaults to *False*
<!-- END schema_docs:switch -->

### Binary sensor

The component can report boiler status on several binary sensors. The *Status* sensors are updated in each message cycle, while the others are only set during initialization, as they are unlikely to change without restarting the boiler.

<!-- BEGIN schema_docs:binary_sensor -->

```yaml

binary_sensor:
  - platform: opentherm
    fault_indication:
      name: fault_indication
    ch_active:
      name: ch_active
    dhw_active: 
      name: dhw_active
    flame_on:
      name: flame_on
    cooling_active: 
      name: cooling_active
    ch2_active:
      name: ch2_active
    diagnostic_indication: 
      name: diagnostic_indication
    dhw_present: 
      name: dhw_present
    control_type_on_off: 
      name: control_type_on_off
    cooling_supported:
      name: cooling_supported
    dhw_storage_tank:
      name: dhw_storage_tank
    master_pump_control_allowed: 
      name: master_pump_control_allowed
    ch2_present:
      name: ch2_present
    dhw_setpoint_transfer_enabled:
      name: dhw_setpoint_transfer_enabled
    max_ch_setpoint_transfer_enabled:
      name: max_ch_setpoint_transfer_enabled
    dhw_setpoint_rw: 
      name: dhw_setpoint_rw
    max_ch_setpoint_rw:
      name: max_ch_setpoint_rw
    service_required:
      name: service_request
    lockout_reset: 
      name: lockout_reset
    gas_fault: 
      name:  Gas/Flame Fault
    low_water_pressure: 
      name: low_water_pressure
    air_fault: 
      name: air_pressure_fault
    water_overtemp: 
      name: water_overtemp    
```
<!-- END schema_docs:binary_sensor -->

### Sensor

The boiler can also report several numerical values, which are available through sensors. Your boiler may not support all of these values, in which case there won't be any value published to that sensor. The following sensors are available:

<!-- BEGIN schema_docs:sensor -->
 ```yaml
sensor:
  - platform: opentherm
    rel_mod_level: 
      name: rel_mod_level
    ch_pressure: 
      name: ch_pressure
    dhw_flow_rate: 
      name: dhw_flow_rate
    t_boiler: 
      name: t_boiler
    t_dhw: 
      name: t_dhw
    t_outside: 
      name: t_outside
    t_ret: 
      name: t_ret
    t_storage: 
      name: t_storage
    t_collector: 
      name: t_collector
    t_flow_ch2: 
      name: t_flow_ch2
    t_dhw2: 
      name: t_dhw2
    t_exhaust: 
      name: t_exhaust
    burner_starts:
      name: burner_starts
    ch_pump_starts: 
      name: ch_pump_starts
    dhw_pump_valve_starts: 
      name: dhw_pump_valve_starts
    dhw_burner_starts: 
      name: dhw_burner_starts
    burner_operation_hours: 
      name: burner_operation_hours
    ch_pump_operation_hours: 
      name: ch_pump_operation_hours
    dhw_pump_valve_operation_hours: 
      name: dhw_pump_valve_operation_hours
    dhw_burner_operation_hours:
      name: dhw_burner_operation_hours
    fault_code:
      id: error 
 ```
<!-- END schema_docs:sensor -->
