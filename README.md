# blinds
Управление горизонтальными жалюзи ( esphome , ESP12F ( 8266) + ULN2003 )
![photo_5357544692272709272_y](https://user-images.githubusercontent.com/64173457/178155504-2fdba7ee-d5f4-4bce-a720-b1113fbacf11.jpg)

Нужен пятивольтовый моторчик с али с драйвером типа ULN2003.
Сперва разбираем чтобы снять штатную крутилку. т.е. снимаем поводки, вытаскиваем плоский вал, удаляем червячный редуктор ( пригодится для других проэктов ).
Лучше всего для крепления вала к моторчику потошли втулки от клемников. Но лучше не крепить жестко, а сделать типа карданного вала, для допусков при сборке.
![photo_5357544692272709246_y](https://user-images.githubusercontent.com/64173457/178155787-761d8396-a5a9-495b-b486-7a98ab0ee4d5.jpg)
Cверлим насквозь, вставляем гвоздь, откусываем, расклепываем.

Вал от продольных смещений тоже лучше закрепить
![photo_5357544692272709247_y](https://user-images.githubusercontent.com/64173457/178155843-3f054602-7967-4073-a4fb-97cb45ba73bf.jpg)

Для контроля положения жалюзи было задействовано два щелевых оптических датчика ( выпаяны из платы принтера )
![datchik-foto](https://user-images.githubusercontent.com/64173457/178155973-6f38ff9b-7635-4585-833e-9f04eea65117.jpeg)

ик фотодиод анодом на плюс 3,3v через резистор ом на 150. катод на землю

Выход датчика - эммиттер на землю, коллектор к gpio. и включить поддтяжку вверх ( или подтянуть резисором на 10к к 3,3v)

На вал приклеиваем/припаиваем/привариваем - два диска. в одном месте сверлим насковозь ( два включенных оптрона будут показывать открытое или горизонтальное состояние жалюзи.  Плюс еще в каждом из дисков делаем по дырке (угол примерно 120 градусов, но лучше определите индивидуально) для определения крайних положений. и собираем что-то подобное ( датчики припаяны к обычной макетке)
![photo_5357544692272709245_y](https://user-images.githubusercontent.com/64173457/178156269-8d53b118-91e5-4e8f-a26b-1fbb8e5979e9.jpg)

Код прошивки:
```
substitutions:
  board_name: blinds
  
esphome:
  name: ${board_name}
  on_boot:
    - priority: -200.0
      then:
      - light.turn_on:
          id: led_1
          brightness: 100%
          effect: 4Pulse
      - switch.turn_on: ${board_name}_auto_stop
      - if: # если жалюзи в неопределенном положении
          condition:
            and:
                - binary_sensor.is_off: close_id
                - binary_sensor.is_off: open_id
          then: 
            - cover.template.publish:
                id: ${board_name}
                position: 0.5 # установка состояния 
                current_operation: IDLE
            - stepper.report_position: # позиция посередине
                id: my_stepper
                position: !lambda return id(my_stepper_global) / 2 ;
            - stepper.set_target: # цель там-же 
                id: my_stepper
                target: !lambda return id(my_stepper_global) / 2 ;
            - cover.open: ${board_name} 
            - while:
                condition:
                  or:
                      - binary_sensor.is_on: close_id
                      - binary_sensor.is_on: open_id
                then:
                    - logger.log: "position set"
                    - delay: 300ms
      - if: # если закрыто
          condition:
            and:
                - binary_sensor.is_on: close_id
                - binary_sensor.is_off: open_id
          then:
            - cover.template.publish:
                id: ${board_name}
                state: CLOSED
                current_operation: IDLE
            - stepper.report_position: # устанавливаем позицию мотора в положение ноль
                id: my_stepper
                position: 0
            - stepper.set_target: # цель позиции мотора тоже ноль
                id: my_stepper
                target: 0
      - if: # Если открыто
          condition:
            and:
                - binary_sensor.is_off: close_id
                - binary_sensor.is_on: open_id
          then: 
            - cover.template.publish:
                id: ${board_name}
                state: OPEN # установка состояния
                current_operation: IDLE
            - stepper.report_position: # установка позиции открытия
                id: my_stepper
                position: !lambda return id(my_stepper_global);
            - stepper.set_target: # цель позиции 
                id: my_stepper
                target: !lambda return id(my_stepper_global);
      - if: # если жалюзи в горизонтальном положении
          condition:
            and:
                - binary_sensor.is_on: close_id
                - binary_sensor.is_on: open_id
          then: 
            - cover.template.publish:
                id: ${board_name}
                position: 0.5 # установка состояния 
                current_operation: IDLE
            - stepper.report_position: # позиция посередине
                id: my_stepper
                position: !lambda return id(my_stepper_global) / 2 ;
            - stepper.set_target: # цель там-же 
                id: my_stepper
                target: !lambda return id(my_stepper_global) / 2 ;

                    


esp8266:
  board: esp01_1m

preferences:
  flash_write_interval: 10080min

globals:
  - id: my_stepper_global #  максимальное количество шагов
    type: int
    restore_value: no
    initial_value: '4210'

wifi:
  ssid: !secret wifi1
  password: !secret password1

captive_portal:

# Disable logging
logger:
#  level: VERY_VERBOSE
  baud_rate: 0

# Enable Home Assistant API
api:
  password: !secret passwordapi

ota:
  password: !secret passwordota

web_server:
  port: 80  



output:
  # вывод встроенного светодиода
  - platform: esp8266_pwm
    id: led
    pin: GPIO02
    frequency: 500 Hz 
    inverted: true

light:
  - platform: monochromatic
    name: LED_1
    output: led
    id: led_1
    internal: true
    effects:
      - pulse:
          name: 1Pulse
      - pulse:
          name: 2Pulse
          transition_length: 0.2s
          update_interval: 0.2s
      - pulse:
          name: 3Pulse
          transition_length: 0.2s
          update_interval: 1.5 s
      - pulse:
          name: 4Pulse
          transition_length: 0.5s
          update_interval: 0.5 s
      - pulse:
          name: 5Pulse
          transition_length: 0.01s
          update_interval: 0.2 s
          

binary_sensor:
  - platform: template
    name: "blinds open"
    lambda: |-
      if (id(open_id).state == true && id(close_id).state == true) {
        return true;
      } else {
        return false;
      }
  - platform: template
    name: "blinds up"
    lambda: |-
      if (id(open_id).state == true && id(close_id).state == false) {
        return true;
      } else {
        return false;
      }
  - platform: template
    name: "blinds down"
    lambda: |-
      if (id(open_id).state == false && id(close_id).state == true) {
        return true;
      } else {
        return false;
      }

  - platform: gpio
    pin:
      number: GPIO13
      mode: INPUT_PULLUP
      inverted: true
    id: window_kitchen
    name: window_kitchen
    device_class: window
    filters:
      - delayed_on: 10ms
      - delayed_off: 10ms
      - delayed_on_off: 10ms
  - platform: status
    name: Status.$board_name
    id: status_id
    internal: true
    on_release:
      then:
        - light.turn_on:
            id: led_1
            brightness: 100%
            effect: 4Pulse
    on_press:
      then:
        - light.turn_off: led_1


  - platform: gpio
    pin:
      number: GPIO05
      mode: INPUT_PULLUP
      inverted: true
    id: open_id
    name: $board_name.open
    filters:
      - delayed_on: 10ms
      - delayed_off: 10ms
      - delayed_on_off: 10ms
    on_press:
      then:
        - logger.log: "open"
        - delay: 300ms
        - if: 
            condition:
              for:
                time: 100ms
                condition:
                  and:
                    - binary_sensor.is_on: close_id
                    - binary_sensor.is_on: open_id
            then:
                - cover.stop: ${board_name}
                - stepper.report_position:
                    id: my_stepper
                    position: !lambda return id(my_stepper_global)/2;
                - stepper.set_target:
                    id: my_stepper
                    target: !lambda return id(my_stepper_global)/2;
                - cover.template.publish:
                    id: ${board_name}
                    position: 0.5
                    current_operation: IDLE

        - if:
            condition:
                - binary_sensor.is_off: close_id
            then:
                - cover.stop: ${board_name}
                - stepper.report_position:
                    id: my_stepper
                    position: !lambda return id(my_stepper_global);
                - stepper.set_target:
                    id: my_stepper
                    target: !lambda return id(my_stepper_global);
                - cover.template.publish:
                    id: ${board_name}
                    position: 1
                    current_operation: IDLE

  - platform: gpio
    pin:
      number: GPIO04
      mode: INPUT_PULLUP
      inverted: true
    id: close_id
    name: $board_name.close
    filters:
      - delayed_on: 10ms
      - delayed_off: 10ms
      - delayed_on_off: 10ms
    on_press:
      then:
        - logger.log: "close"
        - delay: 300ms
        - if:
            condition:
              for:
                time: 100ms
                condition:
                  and:
                    - binary_sensor.is_on: close_id
                    - binary_sensor.is_on: open_id
            then:
                - cover.stop: ${board_name}
                - stepper.report_position:
                    id: my_stepper
                    position: !lambda return id(my_stepper_global)/2;
                - stepper.set_target:
                    id: my_stepper
                    target: !lambda return id(my_stepper_global)/2;
                - cover.template.publish:
                    id: ${board_name}
                    position: 0.5
                    current_operation: IDLE

        - if:
            condition:
                - binary_sensor.is_off: open_id
            then:
                - cover.stop: ${board_name}
                - stepper.report_position:
                    id: my_stepper
                    position: 0
                - stepper.set_target:
                    id: my_stepper
                    target: 0
                - cover.template.publish:
                    id: ${board_name}
                    position: 0
                    current_operation: IDLE

  - platform: gpio
    pin:
      number: GPIO12
      mode:
        input: true
        pullup: true
      inverted: true
    id: ${board_name}_key
    name: ${board_name}_key # кнопка управления
    filters: 
      - delayed_on: 2ms
      - delayed_off: 2ms
      - delayed_on_off: 2ms

    on_multi_click:
    - timing: # двойное нажатие - остановка
        - ON for at most 1s
        - OFF for at most 1s
        - ON for at most 1s
        - OFF for at least 0.2s
      then:
        - logger.log: "Duble Clicked"
        - cover.stop: ${board_name}

    - timing: # долгое нажатие
        - ON for at least 3s
        - OFF for at least 0.2s
      then:
        - logger.log: " Long Clicked"
        - if: # если жалюзи в неопределенном положении
            condition:
              and:
                  - binary_sensor.is_off: close_id
                  - binary_sensor.is_off: open_id
            then: 
              - cover.open: ${board_name} 
              - while:
                  condition:
                    or:
                        - binary_sensor.is_on: close_id
                        - binary_sensor.is_on: open_id
                  then:
                    - logger.log: "position set"
                    - delay: 300ms
                    - cover.open: ${board_name} 
        - if: # если закрыто
            condition:
              and:
                  - binary_sensor.is_on: close_id
                  - binary_sensor.is_off: open_id
            then:
              - cover.open: ${board_name} 
              - delay: 1s
              - while:
                  condition:
                    - binary_sensor.is_off: open_id
                  then:
                    - delay: 1s
                    - if:
                        condition:
                          - binary_sensor.is_on: close_id
                        then:
                          - cover.open: ${board_name}
            else:
              - cover.open: ${board_name}

    - timing: # одиночное нажатие переключение
        - ON for 0.02s to 3s
        - OFF for at least 0.2s
      then:
        - logger.log: "Single Short Clicked"
        - if: #если открываются
            condition:
              lambda: 'return id(${board_name}).current_operation == CoverOperation::COVER_OPERATION_OPENING;'
            then: # То закрываем
                - cover.close: ${board_name} 
                - delay: 2s
                - while:
                    condition:
                      binary_sensor.is_off: close_id
                    then:
                    - delay: 1s
                    - if:
                        condition:
                            - binary_sensor.is_on: open_id
                        then:
                            - cover.close: ${board_name}
            else: #иначе - открываем
              - cover.open: ${board_name}

        - if: # если открыто
            condition:
              and:
                  - binary_sensor.is_on: open_id
            then:
                - cover.close: ${board_name} #закрываем
                - delay: 1s
                - while:
                    condition:
                      binary_sensor.is_off: close_id
                    then:
                    - delay: 1s
                    - if:
                        condition:
                            - binary_sensor.is_on: open_id
                        then:
                            - cover.close: ${board_name}
            else:
                - cover.open: ${board_name}

sensor:
  - platform: wifi_signal
    name: WiFi_Signal_Sensor_$board_name
  - platform: uptime
    name: Uptime_Sensor_$board_name
    id: uptime_sensor
text_sensor:
  - platform: wifi_info
    ip_address:
      name: ESP IP Address.$board_name
    ssid:
      name: ESP Connected SSID.$board_name
    bssid:
      name: ESP Connected BSSID.$board_name
    mac_address:
      name: ESP Mac Wifi Address.$board_name

stepper:
  - platform: uln2003
    id: my_stepper
#    pin_a: GPIO15
#    pin_b: GPIO03
#    pin_c: GPIO16
#    pin_d: GPIO14 # обратное вращение
    pin_a: GPIO14
    pin_b: GPIO16
    pin_c: GPIO03
    pin_d: GPIO15

    max_speed: 250 steps/s
    sleep_when_done: true
  #  step_mode: FULL_STEP # по умолчанию
    step_mode: HALF_STEP
#    step_mode: WAVE_DRIVE
    acceleration: inf # полная скорость сразу
#    acceleration: 200  # ускорение за 200 шагов
    deceleration: inf
#    deceleration: 200  # торможение за 200 шагов

      
button:
  - platform: restart
    name: Reset.$board_name

switch:
  - platform: template
    name: "${board_name} Auto Stop" #выключения контроля концевиков ! опасно, только для отладки
    id: ${board_name}_auto_stop
    optimistic: true
    turn_on_action:
      - logger.log: "Roller Auto Stop on"
    turn_off_action:
      - logger.log: "Roller Auto Stop off"
      - stepper.report_position: # позиция посередине
          id: my_stepper
          position: !lambda return id(my_stepper_global)/2;
      - stepper.set_target: # цель там-же 
          id: my_stepper
          target: !lambda return id(my_stepper_global)/2;

cover:
  - platform: template
    name: "${board_name} blind"
    id: ${board_name}
    has_position: true
    optimistic: true  
    assumed_state: true
    open_action:
      then:
      - if:
          condition:
            or:
              - binary_sensor.is_off: open_id #Если не открыто
              - binary_sensor.is_on: close_id #или закрыто
          then:
          - logger.log: "Opening"
          - stepper.set_target:
              id: my_stepper
              target: !lambda return id(my_stepper_global);
          - light.turn_on:
              id: led_1
              brightness: 100%
              effect: 3Pulse
          - while:
              condition:
                lambda: 'return id(my_stepper).current_position < id(my_stepper_global);'
              then:
                - cover.template.publish:
                    id: ${board_name}
                    position: !lambda 'return (float(float(id(my_stepper).current_position) / id(my_stepper_global)));' 
                    current_operation: OPENING
                - delay: 1000 ms
          - cover.template.publish:
              id: ${board_name}
              state: OPEN
              current_operation: IDLE
    close_action:
      then:
      - if:
          condition:
            or:
              - binary_sensor.is_off: close_id #Если не закрыто
              - binary_sensor.is_on: open_id #или открыто
          then:
          - logger.log: "Closing"
          - stepper.set_target:
              id: my_stepper
              target: 0
          - light.turn_on:
              id: led_1
              brightness: 100%
              effect: 2Pulse
          - while:
              condition:
                lambda: 'return id(my_stepper).current_position > 0;'
              then:
                - cover.template.publish:
                    id: ${board_name}
                    position: !lambda 'return (float(float(id(my_stepper).current_position) / id(my_stepper_global)));' 
                    current_operation: CLOSING
                - delay: 1000 ms
          - cover.template.publish:
              id: ${board_name}
              state: CLOSED
              current_operation: IDLE
    position_action: # установка позиции ползунком
      then:
        - stepper.set_target:
            id: my_stepper
            target: !lambda return int( id(my_stepper_global) * pos);
        - light.turn_on:
            id: led_1
            brightness: 100%
            effect: 1Pulse
        - while:
            condition:
              lambda: 'return id(my_stepper).current_position != int( id(my_stepper_global) * pos);'
            then:
              - cover.template.publish:
                  id: ${board_name}
                  position: !lambda 'return (float(float(id(my_stepper).current_position) / id(my_stepper_global)));' 
              - delay: 1000 ms
        - cover.template.publish:
            id: ${board_name}
            position: !lambda 'return (float(float(id(my_stepper).current_position) / id(my_stepper_global)));' 
            current_operation: IDLE
    stop_action:
      then:
        - stepper.set_target:
            id: my_stepper
            target: !lambda return id(my_stepper).current_position;      
        - logger.log:
            level: DEBUG
            format: 'Stepper %d '
            args: ['id(my_stepper).current_position']
        - light.turn_off:
            id: led_1
        - cover.template.publish:
            id: ${board_name}
            position: !lambda 'return (float(float(id(my_stepper).current_position) / id(my_stepper_global)));' 
            current_operation: IDLE
```
в HA это может выглядеть так:
![Снимок](https://user-images.githubusercontent.com/64173457/178156517-25819eae-6966-46c6-a547-eac11a783d20.PNG)

Карточка:
```
type: vertical-stack
cards:
  - type: entities
    entities:
      - entity: cover.blinds_blind
        name: жалюзи горизонтальные
      - type: custom:slider-entity-row
        entity: cover.blinds_blind
        name: hide_state
        full_row: true
  - type: horizontal-stack
    cards:
      - show_name: true
        show_icon: true
        type: button
        tap_action:
          action: toggle
        entity: binary_sensor.blinds_down
        name: вниз
      - show_name: true
        show_icon: true
        type: button
        tap_action:
          action: toggle
        entity: binary_sensor.blinds_open_2
        name: открыто
      - show_name: true
        show_icon: true
        type: button
        tap_action:
          action: toggle
        entity: binary_sensor.blinds_up
        name: вверх
```
