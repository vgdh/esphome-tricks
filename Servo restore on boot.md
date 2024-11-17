## Restoring the position of the servo and the value of the slider in Home Assistant when the ESPhome has booted.
Use this template as a start point

``` yaml
esphome:
  name: restore-servo-position
  friendly_name:  restore servo position

  on_boot:
    priority: -100
    then:
        - number.set:
            id: number_servo_4
            value: !lambda "return id(saved_position_4);"

globals:
  - id: saved_position_4
    type: int
    initial_value: '11'
    restore_value: true

esp8266:
  board: esp01_1m
  restore_from_flash: true

preferences:
  flash_write_interval: 10s

api:
  encryption:
    key: "----------"


ota:
  - platform: esphome
    password: "---------------"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "ap-esphome"
    password: "jdsfaewrhgjdsyrkjtg"

captive_portal:

servo:
  - id: servo_4
    output: pwm_output4
    min_level: 4.68%
    max_level: 10.06%
    auto_detach_time: 1s
    transition_length: 2s

output:
  - platform: esp8266_pwm
    id: pwm_output4
    pin: GPIO02
    frequency: 50 Hz

number:
  - platform: template
    name: Servo 4
    id: number_servo_4
    min_value: 0
    max_value: 90
    update_interval: 10s 
    lambda:  !lambda "return id(saved_position_4);"
    mode: slider
    step: 1
    set_action:
      then:
        - globals.set:
            id: saved_position_4
            value: !lambda "return x;"
        - servo.write:
            id: servo_4
            level: !lambda 'return (x-45)/45.0;'
```
