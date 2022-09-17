# HWMON - Hardware Monitoring subsystem

*This file represents a preliminary research - there are still some unknowns*

NAS systems are considered autonomous and designed to run in 24x7 operation cycle. This requires a robust hardware 
monitoring in place. DSM over the years contained many different bits here and there. The most basic information like
CPU temperature can be read via SNMP. However, most of the information is actually inaccessible to the users directly, 
or at least not documented.


### What this does to the system?
HWMON itself, as it seems to be called internally, is just one of many hardware-monitoring aspects baked into the OS. 
It seems like HWMON itself doesn't need to be present but the lack of it will cause a sea of errors in the scemd.log 
file. The only required piece looks like to be `(*get_cpu_temperature)(struct _SynoCpuTemp *);` as seen in [mfgBIOS](mfgbios.md).


### Implementation
The HWMON itself is a multipart system. Any user can access its summarized reading by printing files present in 
`/run/hwmon` directory. These files, judging by the system reporting, are written by the `scemd` daemon. The data itself
is sources from the [mfgBIOS](mfgbios.md) via various `hwmon_get_` calls. These calls presumably get the data from 
different places depending on the model.

In order to make a full sense of the data the OS contains a Python script in `/usr/syno/bin/user.data.collector/synouserdata_hw_monitor`
which will parse all files from `/run/hwmon` and present all readings in a more user-friendly form.

```python
#!/usr/bin/python

from subprocess import check_output
from os import listdir
import json
import sys
import os

COLLECTOR_VERSION_KEY = "collector_version"
COLLECTOR_VERSION = 1

HW_MON_PATH="/run/hwmon"
HWMON_CPU_TEMP = "cpu_temperature"
HWMON_THERMEL_SENSOR = "sys_thermal_sensor"
HWMON_VOLTAGE_SENSOR = "sys_voltage_sensor"
HWMON_FAN_SPEED_RPM = "sys_fan_speed_rpm"
HWMON_HDD_BACKPLANE_STATUS = "hdd_backplane_status"
HWMON_PSU_STATUS = "psu_status"
HWMON_CURRENT_SENSOR = "sys_current_sensor"
HW_MON_LIST=[HWMON_CPU_TEMP, HWMON_THERMEL_SENSOR, HWMON_VOLTAGE_SENSOR, HWMON_FAN_SPEED_RPM, HWMON_HDD_BACKPLANE_STATUS, HWMON_PSU_STATUS, HWMON_CURRENT_SENSOR]

HWMON_CPU_TEMP_NAME = "CPU_Temperature"
HWMON_SYS_THERMAL_NAME = "System_Thermal_Sensor"
HWMON_SYS_VOLTAGE_NAME = "System_Voltage_Sensor"
HWMON_SYS_FAN_RPM_NAME = "System_Fan_Speed_RPM"
HWMON_PSU1_STATUS_NAME = "PSU_1_Status"
HWMON_PSU2_STATUS_NAME = "PSU_2_Status"
HWMON_HDD_BP_STATUS_NAME = "HDD_Backplane_Status"
HWMON_SYS_CURRENT_NAME = "System_Current_Sensor"

THERMEL_SENSOR_VALID_FIELD_LIST = ["Remote1", "Local", "Remote2", "system", "ADT1 Local", "ADT2 Local"]
VOLTAGE_SENSOR_VALID_FIELD_LIST = ["VCC", "VPP", "V33", "V5", "V12", "ADT1 V33", "ADT2 V33"]
HDD_BACKPLANE_VALID_FIELD_LIST = ["hdd_detect", "hdd_enable"]
PSU_STATUS_VALID_FIELD_LIST = ["power_in", "power_out", "temperature_1", "temperature_2", "temperature_3", "fan_speed", "status", "fan_voltage"]
CURRENT_SENSOR_VALID_FIELD_LIST = ["ADC"]

def checkHwMonitorSupport(name):
    if os.path.isfile(HW_MON_PATH + "/" + name + ".json"):
        return True
    else:
        return False

def getHwMonitorJson(name):
    try:
        with open(HW_MON_PATH + "/" + name + ".json") as hw_mon_file:
            return json.load(hw_mon_file)
    except Exception as e:
        return False

def checkValidField(origin_json, valid_list):
    new_json={}
    for key, value in origin_json.items():
        if(key in valid_list):
            new_json[key] = origin_json[key]
    return new_json

def convertValueToInt(origin_json):
    for key, value in origin_json.items():
        origin_json[key]=int(value)
    return origin_json

def convertHexValueToInt(origin_json):
    for key, value in origin_json.items():
        try:
            tmp_value=int(value, base=16)
            origin_json[key]=tmp_value
        except Exception as e:
            continue
    return origin_json

def main():
    result = {
        COLLECTOR_VERSION_KEY: COLLECTOR_VERSION,
        HWMON_CPU_TEMP : "",
        HWMON_THERMEL_SENSOR : "",
        HWMON_VOLTAGE_SENSOR : "",
        HWMON_FAN_SPEED_RPM : "",
        HWMON_HDD_BACKPLANE_STATUS : "",
        HWMON_PSU_STATUS : [],
        HWMON_CURRENT_SENSOR : "",
    }
    for hw_mon_item in HW_MON_LIST:
        if checkHwMonitorSupport(hw_mon_item):
            temp_json = getHwMonitorJson(hw_mon_item)
            if False == temp_json:
                continue
            if HWMON_CPU_TEMP == hw_mon_item:
                result[HWMON_CPU_TEMP] = []
                sub_tmp_json=temp_json[HWMON_CPU_TEMP_NAME]
                for key, value in sub_tmp_json.items():
                    result[HWMON_CPU_TEMP].append(int(value))
            elif HWMON_THERMEL_SENSOR == hw_mon_item:
                target_temp_json=temp_json[HWMON_SYS_THERMAL_NAME]
                target_temp_json=checkValidField(target_temp_json, THERMEL_SENSOR_VALID_FIELD_LIST)
                result[HWMON_THERMEL_SENSOR] = convertValueToInt(target_temp_json)
            elif HWMON_VOLTAGE_SENSOR == hw_mon_item:
                target_temp_json=temp_json[HWMON_SYS_VOLTAGE_NAME]
                target_temp_json=checkValidField(target_temp_json, VOLTAGE_SENSOR_VALID_FIELD_LIST)
                result[HWMON_VOLTAGE_SENSOR] = convertValueToInt(target_temp_json)
            elif HWMON_FAN_SPEED_RPM == hw_mon_item:
                result[HWMON_FAN_SPEED_RPM] = []
                for key, value in temp_json[HWMON_SYS_FAN_RPM_NAME].items():
                    result[HWMON_FAN_SPEED_RPM].append(int(value))
            elif HWMON_HDD_BACKPLANE_STATUS == hw_mon_item:
                target_temp_json=temp_json[HWMON_HDD_BP_STATUS_NAME]
                target_temp_json=checkValidField(target_temp_json, HDD_BACKPLANE_VALID_FIELD_LIST)
                result[HWMON_HDD_BACKPLANE_STATUS] = convertValueToInt(target_temp_json)
            elif HWMON_PSU_STATUS == hw_mon_item:
                target_temp_json=temp_json[HWMON_PSU1_STATUS_NAME]
                target_temp_json=checkValidField(target_temp_json, PSU_STATUS_VALID_FIELD_LIST)
                result[HWMON_PSU_STATUS].append(convertHexValueToInt(target_temp_json))

                target_temp_json=temp_json[HWMON_PSU2_STATUS_NAME]
                target_temp_json=checkValidField(target_temp_json, PSU_STATUS_VALID_FIELD_LIST)
                result[HWMON_PSU_STATUS].append(convertHexValueToInt(target_temp_json))
            elif HWMON_CURRENT_SENSOR == hw_mon_item:
                target_temp_json=temp_json[HWMON_SYS_CURRENT_NAME]
                target_temp_json=checkValidField(target_temp_json, CURRENT_SENSOR_VALID_FIELD_LIST)
                result[HWMON_CURRENT_SENSOR] = convertValueToInt(target_temp_json)
        else:
            if HWMON_CPU_TEMP == hw_mon_item or HWMON_FAN_SPEED_RPM == hw_mon_item or HWMON_PSU_STATUS == hw_mon_item:
                result[hw_mon_item] = []
            else:
                result[hw_mon_item] = {}
    sys.stdout.write(json.dumps(result))

if __name__ == '__main__':
    main()
```

#### Sensor types
[mfgBIOS](mfgbios.md) operations list gives a hint about the list of *type* of sensors supported by the HWMON subsystem:
 - FAN speed (`(*hwmon_get_fan_speed_rpm)(SYNO_HWMON_SENSOR_TYPE *)`)
 - Power supply status (`(*hwmon_get_psu_status)(SYNO_HWMON_SENSOR_TYPE *, int)`)
 - Voltage reading (`(*hwmon_get_sys_voltage)(SYNO_HWMON_SENSOR_TYPE *)`)
 - Hard drive backplane status (`(*hwmon_get_backplane_status)(SYNO_HWMON_SENSOR_TYPE *)`)
 - Temperature (`(*hwmon_get_sys_thermal)(SYNO_HWMON_SENSOR_TYPE *)`)
 - Current/power draw (`(*hwmon_get_sys_current)(SYNO_HWMON_SENSOR_TYPE *)`)

Within each sensor type there can be, at least on DSM v6.2, at most 10 sensors (see `MAX_SENSOR_NUM` in `synobios.h` 
included with GPLed kernel sources).


#### Sensors identification
As each type contains multiple sensors there's a need for their identification. When [mfgBIOS](mfgbios.md) is asked to
provide the reading the following structure is passed to it via ioctl:

```
//Source: include/linux/synobios.h
typedef struct _SYNO_HWMON_SENSOR {
    char sensor_name[MAX_SENSOR_NAME];
    char value[MAX_SENSOR_VALUE];
} SYNO_HWMON_SENSOR;

typedef struct _SYNO_HWMON_SENSOR_TYPE {
    char type_name[MAX_SENSOR_NAME];
    int sensor_num;
    SYNO_HWMON_SENSOR sensor[MAX_SENSOR_NUM];
} SYNO_HWMON_SENSOR_TYPE;
```

One of the drivers in the kernel source - `drivers/hwmon/adt7475.c` - provides a good example as to which values are 
expected to be an "input" and which are the "output".

 - `type_name`: we weren't able to determine the exact purpose of `type_name` (but it's an output value) as it doesn't
                seem to be set by the caller nor is set by the `adt7475.c` driver. However, the `synobios.h` file 
                contains a list of constants for each type (but just one for each type). See next section.
 - `sensor_num`: number of expected elements in `sensor` array. Value of this field is determined by the platform and
                 if external source is "asked" for the value (e.g. `adt7475.c`) that value is already determined. This 
                 ensured drivers don't need to care about number of sensors or their order - it will be predetermined
                 by the mfgBIOS
 - `sensor`: readings from sensors
   - `sensor_name`: name/code of the sensor being read, see below for details. This name, if external source is called
                    (e.g. `adt7475.c`) is already filled in. Things calling ioctl to the mfgBIOS ask without this name
                    populated and can expect the name to be populated. This lets us conclude that mfgBIOS is the one
                    populating sensor names.
   - `value`: unitless reading of the sensor


#### Known sensors
The list of sensors within each type is finite and predefined. The `include/linux/synobios.h` gives us both the 
categories/types as well as sensors within each type. Below we presented them in a tree-like format with `HWMON_` 
constants listed alongside the actual codenames in the mfgBIOS order. Names should be self-explanatory to determine what 
they mean (good design!).

 - `CPU_Temperature` (`HWMON_CPU_TEMP_NAME`)
   - example: `CPU_#` where `#` represents 0-n integer for the CPU core; value is in degrees celsius
 - `System_Fan_Speed_RPM` (`HWMON_SYS_FAN_RPM_NAME`)
   - example: `fan1_rpm` (`HWMON_SYS_FAN1_RPM`); value is in RPM (not tach units!)
   - `synobios.h` defines them up to `fan6_rpm` (`HWMON_SYS_FAN6_RPM`) but providing 7th one seem to work with the 
     `synouserdata_hw_monitor`. Seeing as `MAX_SENSOR_NUM` is 10 we can assume the structure is prepared to handle 11
     fans.
   - Interestingly v7 `synobios.h` removes `HWMON_SYS_FAN5_RPM` and `HWMON_SYS_FAN6_RPM`
 - `PSU_%d_Status` (`HWMON_PSU_STATUS_NAME`)
   - the name itself is a `sprintf`-complaint pattern name. However, `synobios.h` provides the only two valid values of
     `PSU_1_Status` (`HWMON_PSU1_STATUS_NAME`) and `PSU_2_Status` (`HWMON_PSU2_STATUS_NAME`)
   - `power_in` (`HWMON_PSU_SENSOR_PIN`): units are unknown, presumably watts
   - `power_out` (`HWMON_PSU_SENSOR_POUT`): units are unknown, presumably watts
   - `temperature` (`HWMON_PSU_SENSOR_TEMP`): PSU internal temperature in degrees celsius; this doesn't seem to work 
      with the v7 conversion script but does in v6.2. However, using `temperature_1` - `temperature_3` does the job. 
      These numbered temperatures are available since v7 (defined as `HWMON_PSU_SENSOR_TEMP1` - 
      `HWMON_PSU_SENSOR_TEMP3`).
   - `fan_speed` (`HWMON_PSU_SENSOR_FAN`): value is in RPM (not tach units!)
   - `status` (`HWMON_PSU_SENSOR_STATUS`): unknown; presumably something like OK/NOK; it's a hex value
   - `fan_voltage` (`HWMON_PSU_SENSOR_FAN_VOLT`): v7 only, presumably internal PSU fan voltage
 - `System_Voltage_Sensor` (`HWMON_SYS_VOLTAGE_NAME`)
   - Kernel doesn't provide a list of voltages supported. However, the conversion script in v7 lists the valid values, 
     where most are self-explanatory.
   - `VCC`: presumably input voltage to the system
   - `VPP`: presumably peak-to-peak (min-max?) voltage to the system
   - `V33`: 3.3V rail voltage
   - `V5`: 5V rail voltage
   - `V12`: 12V rail voltage
   - `ADT1 V33`: some first unknown 3.3V rail voltage (adt = additional?)
   - `ADT2 V33`: some second unknown 3.3V rail voltage
 - `HDD_Backplane_Status` (`HWMON_HDD_BP_STATUS_NAME`)
   - `hdd_detect` (`HWMON_HDD_BP_DETECT`): unknown, presumably number of detected disks
   - `hdd_enable` (`HWMON_HDD_BP_ENABLE`): unknown, presumably number of active/running disks
   - `hdd_intf` (`HWMON_HDD_BP_INTF`): v7 only, appears unused?
 - `System_Thermal_Sensor` (`HWMON_SYS_THERMAL_NAME`)
   - Kernel doesn't provide a hint as to which temperatures are specified, however the conversion script does contain 
     the list of sensors which are valid and thus can be used to make a compatibility layer. We don't have a device to
     test the meaning of each of the temperatures nor there seems to be a way to see them in any GUI.
 - `System_Current_Sensor` (`HWMON_SYS_CURRENT_NAME`)
   - Kernel doesn't provide any information about valid sensors for this type.


### How does it play with loaders?
A loader should provide values to the `hwmon` subsystem. Ideally this should happen in the kernel so that hardware 
values are presented to all userland tools (e.g. `scemd`) as they were real.
