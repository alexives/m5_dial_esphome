## M5 Dial ESPHome

An [ESPHome](https://esphome.io/) configuration for the [M5Stack Dial](https://shop.m5stack.com/products/m5stack-dial-esp32-s3-smart-rotary-knob-w-1-28-round-touch-screen).

Based largely on [dgaust's m5 dial esphome files](https://github.com/dgaust/esphome_files) and also [this home assistant community thread](https://community.home-assistant.io/t/m5stack-dial-esp32-s3-smart-rotary-knob/623518).

To use, you can use the example configuration to set up your dial with various components. For the weather config, you'll also need to set up forecast entities.

### Example ESPHome Config:

```yaml
esphome:
  name: bathroom-dial

substitutions:
  # Always required
  room: Bathroom
  return_to_default_page: 20s # This is the delay it uses to return to the default page
  encryption_key: !secret encryption_key # Whatever you want to use for the encryption key
  ota_password: !secret ota_password # OTA password to keep folks from ota'ing you
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password
  fallback_password: !secret fallback_password
  default_page: WeatherPage # This is the page it defaults to
  secondary_page: MusicPage # This page is displayed when the backlight button is pressed
  # Media player input selectors/icons
  media_player_control: media_player.bathroom # This is the media player controlled by MusicPage
  script_delay: 1s # This is the delay after adjusting the rotary before it sends the updated volumef
  source_1: MPR News
  source_1_image: mdi:newspaper
  source_2: The Current
  source_2_image: mdi:guitar-electric
  source_3: "Shut up, it's fun 🍺"
  source_3_image: mdi:glass-mug-variant
  # Weather related options
  weather_entity: weather.pirateweather # This is the weather entity it uses to get current temperature
  forecast_entity: sensor.weather_forecast_daily # This is the entity defined by the template sensor below

packages:
  m5dial:
    url: https://gitlab.com/alexives/m5_dial_esphome
    ref: main
    files: 
      # These files are all optional and you can remove them if you want.
      - music.yaml # Adds the display page "MusicPage". It switches to this display when you rotate the dial and adjusts the volume
      - weather.yaml # Displays the weather based on the forecast entity
      # The following files are always required
      - common.yaml
      - colors.yaml
      - fonts.yaml
    refresh: 60sec

display: # Control Page Changes
  - id: !extend dial_display
    on_page_change:
      - then:
          - delay: $return_to_default_page
          - display.page.show: $default_page

binary_sensor: # Control how the bezel button works
  - id: !extend bezel_button
    on_press:
      - if:
          condition:
            display.is_displaying_page: $default_page
          then:
            - display.page.show: $secondary_page
          else:
            - display.page.show: $default_page

script:
  - id: !extend music_before_script
    then:
      - homeassistant.service:
          service: media_player.unjoin
          data_template:
            entity_id: $media_player_control
```

You'll also need the following template sensor for the weather page:

```yaml
template:
  - trigger:
      - platform: time_pattern
        hours: /1
    action:
      - service: weather.get_forecasts
        data:
          type: daily
        target:
          entity_id: weather.pirateweather
        response_variable: daily
    sensor:
      - name: Weather Forecast daily
        unique_id: weather_forecast_daily
        state: "{{ now().isoformat() }}"
        attributes:
          forecast: "{{ daily['weather.pirateweather'].forecast[:2] }}"
          temperature: "{{ daily['weather.pirateweather'].forecast[0].temperature }}"
          templow: "{{ daily['weather.pirateweather'].forecast[0].templow }}"
          precipitation_probability: "{{ daily['weather.pirateweather'].forecast[0].precipitation_probability }}"
```

### Package Files

Packages typically require 2 different things they need in order to function:
- **File to include**
  The included file goes in the `packages:` -> `m5dial:` -> `files:` section.
- **Required substitutions**
  Required substitutions are used to populate various entities or variables

They also provide 2 things
- Named Display Page
  For example, music provides `MusicPage` that can be referenced in `default_page`, or used in scripts/actions.
- Script extensions
  For example, the music buttons provide `music_before_script` that runs before selecting a source. This can be useful for actions
  like calling unjoin to isolate a media player if desired.

#### Music

This is designed to control a music player, it provides a volume adjustment, pause/skip/back, and 3 buttons to quick play a few sources.

**File:** `music.yaml`
**Page Name:** `MusicPage`

| Substitution | Description | Example | Required |
| ------------ | ----------- | ------- | -------- |
| media_player_control | A media player entity to conrol | `media_player.bathroom` | **Yes** |
| source_1 | The source to select from the top button on the control page | `MPR News` | **Yes** |
| source_1_image | The material design icon you'd like to use | `mdi:newspaper` | **Yes** |
| source_2 | The source to select from the left button on the control page | `The Current` | **Yes** |
| source_2_image | The material design icon you'd like to use | `mdi:guitar-electric` | **Yes** |
| source_3 | The source to select from the bottom button on the control page | `"Shut up, it's fun 🍺"` | **Yes** |
| source_3_image | The material design icon you'd like to use | `mdi:glass-mug-variant` | **Yes** |

##### Script: `music_before_script`

This script runs before all of the buttons that trigger a source.

Example:
```yaml
script:
  - id: !extend music_before_script
    then:
      - homeassistant.service:
          service: media_player.unjoin
          data_template:
            entity_id: $media_player_control
```

#### Weather

**File:** `weather.yaml`
**Page Name:** `WeatherPage`

| Substitution | Description | Example | Required |
| ------------ | ----------- | ------- | -------- |
| scale_entity | A weather entity with the current temperature and conditions | weather.pirateweather | **Yes** |
| forecast_entity | A template sensor using the [template below](#required-forecast-template-sensor) | sensor.weather_forecast_daily | **Yes** |

##### Required Forecast Template Sensor

```yaml
template:
  - trigger:
      - platform: time_pattern
        hours: /1
    action:
      - service: weather.get_forecasts
        data:
          type: daily
        target:
          entity_id: weather.pirateweather
        response_variable: daily
    sensor:
      - name: Weather Forecast daily
        unique_id: weather_forecast_daily
        state: "{{ now().isoformat() }}"
        attributes:
          forecast: "{{ daily['weather.pirateweather'].forecast[:2] }}"
          temperature: "{{ daily['weather.pirateweather'].forecast[0].temperature }}"
          templow: "{{ daily['weather.pirateweather'].forecast[0].templow }}"
          precipitation_probability: "{{ daily['weather.pirateweather'].forecast[0].precipitation_probability }}"
```

#### Scale

This takes a bathroom scale in KG and converts it to Lbs on the dial. Helpful if you have some folks who like one or the other!

**File:** `scale.yaml`
**Page Name:** `ScalePage`

| Substitution | Description | Example | Required |
| ------------ | ----------- | ------- | -------- |
| scale_entity | A sensor entity that provides a mass in kilograms | sensor.bathroom_scale | **Yes** |

No scripts to override