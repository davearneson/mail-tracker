# Mailbox Sensor

![Mailbox Sensor](https://i.imgur.com/V2apQt4.jpg)

This project creates a sensor that tracks what time the mailbox is opened and sends an actionable alert which gives me the option to reset it. I have no programming experience, so I would not be surprised if programming best practices are not in play here, and for that I ask your forgiveness. I’m sure there are simpler ways to accomplish this and or streamline, and I’d love to hear them! I’m sharing because of how much reading about other people’s projects has helped me to learn.

This sensor uses a Wyze sense door sensor to trigger the automation when the mailbox is opened, but any door sensor can be used. I set up the the Wyze sensors using this integration: (https://github.com/kevinvincent/ha-wyzesense)

## Home Assistant Helpers

I used HA’s helpers to set up an input_boolean entity and an input_datetime entity. 

 I created the input_boolean entity in the UI by going to Configuration/Helpers and clicking the add button, then “Toggle.” I then added a name, in my case “You’ve Got Mail”. This entity will be used to mark the mail as arrived. I didn’t want to use just the Wyze door sensor, because the mailbox is typically opened and then immediately closed, so I needed a switch that would stay “on” until turned off manually or by an automated reset.

I created the input_datetime entity in the UI by going to Configuration/Helpers and clicking the add button, then “Date and/or time.” I then added a name, in my case “Mailbox Time Arrived”. This entity will be used to set the time that mail was delivered. 

# configuration.yaml

There are two sections that have to be added to configuration.yaml for this tracker:

## Template Sensors

First I created a youve_got_mail template sensor which is outputting text based on the state of the input_boolean entity that I created using Helpers. I will create an automation later that will tie the input_boolean entity to the Wyze door sensor, but this is the main component of the mailbox sensor. You will also need to enable the time_date sensor if you have not already done so.

```
# configuration.yaml

##########Mailbox Sensor##########
sensor:
- platform: template
  sensors:
    youve_got_mail:
        value_template: >
            {% if is_state('input_boolean.you_ve_got_mail', 'on') %}
                You've got mail!
            {% else %}
                No mail today.
            {% endif %}

- platform: time_date
  display_options:
      - 'time'
      - 'date'
      - 'date_time'
      - 'date_time_utc'
      - 'date_time_iso'
      - 'time_date'
      - 'time_utc'
      - 'beat'
```
## Actionable Notifications

This part of the project requires that you have set up actionable notifications. I had some trouble doing this when I first did it for another project, but it’s fairly well documented here: https://companion.home-assistant.io/docs/notifications/actionable-notifications/

![Mailbox Opened Actionable Notification](https://i.imgur.com/dLJX5vR.jpg)

```
# configuration.yaml

ios:
  push:
    categories:
      - name: Mailbox Alert
        identifier: 'mailboxalert'
        actions:
          - identifier: 'MAILBOX_RESET'
            title: 'It was me.'
```
# Automations

I created three automations for the main part of this project, and I’ll do my best to explain them and the logic behind them: (I also have a tts alert through Alexa Media Player, but I’m not going to cover that in this writeup.)

## Automation Linking Wyze sensor to input_boolean entity and Sending Actionable Notification

When the Wyze sensor is opened, this automation checks to make sure that a) the automation hasn’t been fired in the last 2 minutes, and b) the input_boolean entity is “off,” and if either thing is true, it turns on the input_boolean.you_ve_got_mail entity, and then sends an actionable notification (“Mailbox has been opened.”) to an iOS device with the option to reset the sensor. (“It was me”). I have the conditions set so I don’t get a bunch of notifications if it opens a couple of times in a row. (Sometimes my wife and I will both check out of habit when we get home together, or sometimes the mail carrier will open it a couple of times to put multiple things in, etc.)

```
# automations.yaml

- alias: Mail Notification
  description: ''
  trigger:
  - entity_id: binary_sensor.wyzesense_778347c9
    from: 'off'
    platform: state
    to: 'on'
  condition:
  - condition: or
    conditions:
    - condition: template
      value_template: '{{ ( as_timestamp(now()) - as_timestamp(state_attr(''automation.mail_notification'', ''last_triggered'')) |int(0) ) > 120 }}'
    - condition: state
      entity_id: input_boolean.you_ve_got_mail
      state: 'off'
  action:
  - data:
        data:
            push:
                category: mailboxalert
      message: Mailbox has been opened.
    service: notify.mobile_app_iphone
  - data: {}
    entity_id: input_boolean.you_ve_got_mail
    service: input_boolean.turn_on
```

## Automation Resetting Mail Sensor Each Day or When Reset From Actionable Notification

This automation resets the input_boolean.you_ve_got_mail entity to “off” every day at 8:30 a.m. or when it is reset through the actionable notification.

```
# automations.yaml

- alias: Mail Notification - Reset Mail Sensor Each Day
  description: ''
  trigger:
  - at: 08:30:00
    platform: time
  - event_data:
      actionName: MAILBOX_RESET
    event_type: ios.notification_action_fired
    platform: event
  condition: []
  action:
  - data: {}
    entity_id: input_boolean.you_ve_got_mail
    service: input_boolean.turn_off
```

## Automation Setting the Time the Mail Arrived

I added this automation to keep track of the time the mail arrived. If the mailbox is opened, and no one resets it (“It was me”) for two minutes, then this automation will set the input_datetime entity I created with Helpers to two minutes ago. (The time the mailbox was opened). This is accomplished by converting the current time to a timestamp, subtracting 120 seconds, and then reconverting the timestamp to a format the input_datetime entity understands.

```
# automations.yaml

- alias: Mail Notification - Set Time Mail Arrived
  description: ''
  trigger:
  - entity_id: sensor.youve_got_mail
    for: 00:02:00
    platform: state
    to: You've got mail!
  condition: []
  action:
  - data_template:
      datetime: '{{ (as_timestamp(states.sensor.date_time_iso.state) - 120)|timestamp_custom("%Y-%m-%d
        %H:%M:%S") }}'
    entity_id: input_datetime.mailbox_time_arrived
    service: input_datetime.set_datetime
```

# Custom Card for Lovelace UI

I combined everything into a simple custom picture elements card. I had originally used an entity card with three button cards combined into stacks, but found I preferred the look of everything unified together, so I came up with this:

![Mailbox Sensor](https://i.imgur.com/V2apQt4.jpg)

```
elements:
  - entity: sensor.youve_got_mail
    style:
      bottom: 15%
      color: black
      font-size: 26px
      left: 1.5%
      transform: initial
    type: state-label
  - entity: input_datetime.mailbox_time_arrived
    icon: 'mdi:mailbox'
    style:
      color: 'rgb(68,115,158)'
      left: 83%
      top: 30%
      transform: 'scale(3,3)'
    tap_action:
      action: more-info
    type: state-icon
image: 'https://i.imgur.com/P7QdVCU.png'
type: picture-elements
```

![Mailbox Time Arrived](https://i.imgur.com/4fuMpoK.jpg)

If you click or tap on the mailbox icon on the lovelace card, it will open the Mailbox Time Arrived entity, and show you the date and time the mailbox was last opened without being reset.
