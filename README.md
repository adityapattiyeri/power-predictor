# power-predictor

I have setup a fully-fledged home assistant setup at my parent’s house with energy monitoring and everything. I noticed that my dad sometimes forgets to turn off the induction cook top. So I decided create a couple of automations and sensors to detect if the induction has been left on. It is kinda quirky, but it kinda gets the job done. I am sure that are better ways to get this done and I'm completely open to suggestion.


I am using a simple pulse meter that measures the power consumption of the entire house. It’s quite sensitive and the readings are fairly accurate. The Bayesian sensors can be improved by including time of day, motion sensors, etc.., but this will do for a proof of concept.

So here is how it works:

1. I created a Bayesian sensor(binary_sensor.idle_load) that detects when there are no high power devices running.
```
- name: "Idle Load"
  platform: "bayesian"
  prior: 0.5
  probability_threshold: 0.9
  observations:
    - platform: template
      value_template: >
        {{(((states.sensor.grid_power.state|float)+(states.sensor.solar_power.state|float))<0.8)}}
      prob_given_true: 0.95 
```

2. Then a automation runs every 3 minutes to update an input boolean(input_number.idle_power) while there are no high power devices running.
```
alias: 'power prediction: idle load'
description: ''
trigger:
  - platform: time_pattern
    minutes: /1
condition:
  - condition: state
    entity_id: binary_sensor.idle_load
    state: 'on'
action:
  - service: input_number.set_value
    data_template:
      entity_id: input_number.idle_power
      value: >-
        {{ (states.sensor.grid_power.state|float) }}
mode: single
```
3. Then there is a second Bayesian sensor(binary_sensor.induction) that actually detects the induction cook top.
```
- name: "Induction"
  platform: "bayesian"
  prior: 0.5
  probability_threshold: 0.9
  observations:
    - platform: template
      value_template: >
        {{ 
          (((states.sensor.grid_power.state|float)+(states.sensor.solar_power.state|float)-(states.input_number.idle_power.state|float)) < 1)
          and
          (((states.sensor.grid_power.state|float)+(states.sensor.solar_power.state|float)-(states.input_number.idle_power.state|float)) > 0.8)
        }} 
      prob_given_true: 0.95  
}
```

This binary sensor(binary_sensor.induction) can be used to trigger an automation that sends a notification to the google hub if its left on for more that 10 mins, but I kinda ran into a problem here. The induction cook top pulse width modulates the power and doesn’t stay on the entire time. So I had to do a workaround to get it working.

4. I used node red to create a "filter" that essentially waits until the binary sensor(binary_sensor.induction) to turn off for a couple of seconds before sending the off signal. It kinda works like a "delayed_off" filter in Esphome. This filtered value is the used to toggle another input boolean(input_boolean.induction_notify). 
![Screenshot 2021-11-03 at 9 30 28 PM](https://user-images.githubusercontent.com/61015809/140096848-3d3342a8-7af5-4055-9870-3841e99b649c.png)

5.This input boolean(input_boolean.induction_notify) is used to trigger the notification automation.
```
alias: 'power prediction: induction warning'
description: ''
trigger:
  - platform: state
    entity_id: input_boolean.induction_notify
    from: 'off'
    to: 'on'
    for:
      hours: 0
      minutes: 10
      seconds: 0
      milliseconds: 0
condition: []
action:
  - service: notify.family
    data:
      title: The induction might be left on!
      message: You might want to check!
  - service: media_player.play_media
    target:
      entity_id:
        - media_player.foyer_speaker
        - media_player.kitchen_display
    data:
      media_content_id: http://192.168.0.10:8123/local/voices/induction_warning.mp3
      media_content_type: audio/mp3
mode: single

```
