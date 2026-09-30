esphome:
  name: kitchen-vent-diff-pressure
  friendly_name: Kitchen ventilation diff pressure
  on_boot:
    priority: -100
    then:
      - script.execute: sensor_polling_logic

script:
  - id: sensor_polling_logic # Wait on start. May be helpful for update
    mode: restart
    then:
      - delay: 5s
      - while:
          condition:
            # This makes the loop run forever
            lambda: 'return true;'
          then:
            - component.update: differential_pressure_raw
            - delay: 100ms
            
esp32:
  board: lolin_s2_mini
  variant: ESP32S2
  framework:
    type: arduino


# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "****************"

ota:
  - platform: esphome
    password: "**************************"
    on_begin:
      then:
        - script.stop: sensor_polling_logic
        - logger.log: "OTA starting, stopped sensor polling for stability."
          
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  fast_connect: True
  min_auth_mode: WPA2
  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Kitchen-Vent-Diff-Pressure"
    password: "*****************************"

  manual_ip:
    static_ip: 172.16.84.15
    gateway: 172.16.84.1
    subnet: 255.255.255.0

captive_portal:
    
globals:
  # Array to store the last 5 readings
  - id: diff_pressure_avg_history
    type: float[10]
    initial_value: '{0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0}'
  
  # Index pointer for the circular buffer
  - id: buffer_index
    type: int
    initial_value: '0'
    
# Global variable to hold the value from 5 seconds ago
  - id: diff_pressure_avg_delayed_value
    type: float
    initial_value: '0.0'
    
# light:
#   - platform: status_led
#     name: "Status LED"
#     id: esp_status_led
#     icon: "mdi:alarm-light"
#     restore_mode: ALWAYS_OFF
#     pin:
#       number: GPIO15

i2c:
  sda: GPIO3
  scl: GPIO37
  scan: true


# debug:
#   update_interval: 1s

sensor:
  - platform: internal_temperature
    name: "ESP32 Temperature"
    update_interval: 5s
    filters: 
      - delta: 0.5
          

  # DEBUG #############################
  # - platform: uptime
  #   name: "ESP Uptime"

  # - platform: debug
  #   free:
  #     name: "Free Heap"
  #   loop_time:
  #     name: "ESP Loop Time"
  #   fragmentation: 
  #     name: "Fragmentation"
  #####################################


  - platform: sdp3x
    id: differential_pressure_raw
    address: 0x25
    measurement_mode: differential_pressure
    unit_of_measurement: "hPa"
    update_interval: never # Managed by script below

  - platform: copy
    source_id: differential_pressure_raw
    name: "Differential Pressure"
    unit_of_measurement: "Pa"
    accuracy_decimals: 2
    filters:
      - sliding_window_moving_average:
          window_size: 10
          send_every: 10
      - lambda: return x * 100.0;
      - round: 2

  - platform: copy
    source_id: differential_pressure_raw
    id: diff_pressure_avg_long
    internal: true
    unit_of_measurement: "hPa"
    filters:
      - sliding_window_moving_average:
          window_size: 600
          send_every: 10
    on_value:
      then:
        - lambda: |-
            // 1. The value currently at our pointer is the oldest one
            id(diff_pressure_avg_delayed_value) = id(diff_pressure_avg_history)[id(buffer_index)];
            
            // 2. Overwrite that oldest slot with the brand new incoming value (x)
            id(diff_pressure_avg_history)[id(buffer_index)] = x;
            
            // 3. Advance the pointer to the next slot (wrap around 0-number of elements)
            id(buffer_index) = (id(buffer_index) + 1) % 10;
                      
  - platform: copy
    source_id: differential_pressure_raw
    id: differential_pressure_avg_now
    internal: true
    unit_of_measurement: "hPa"
    on_value:
      then:
        - component.update: differential_pressure_delta
    filters:
      - sliding_window_moving_average:
          window_size: 10
          send_every: 10


  - platform: template
    name: "Differential Pressure Delta"
    id: differential_pressure_delta
    unit_of_measurement: "Pa"
    accuracy_decimals: 2
    lambda: |-
      if (id(diff_pressure_avg_delayed_value) == 0.0 || isnan(id(diff_pressure_avg_long).state)) {
        return NAN;
      }
      return id(differential_pressure_avg_now).state - id(diff_pressure_avg_delayed_value);
    update_interval: never
    filters:
      - lambda: return x * 100.0;
      - round: 2
