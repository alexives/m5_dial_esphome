## M5 Dial ESPHome

This is a configuration for managing m5 dials.

To use, you can use the example configuration to set up your dial with various components. For the weather config, you'll also need to set up forecast entities.

Example ESPHome Config:

```yaml
esphome:
  name: bathroom-dial

substitutions:
  room: Bathroom
  default_page: WeatherPage # This is the page it defaults to
  secondary_page: MusicPage # This page is displayed when the backlight button is pressed
  media_player_control: media_player.bathroom # This is the media player controlled by MusicPage
  scale_entity: sensor.mi_body_composition_scale_61dc_mass # This is the scale used by ScalePage
  script_delay: 1s # This is the delay after adjusting the rotary before it sends the updated volumef
  # Media player input selectors/icons
  source_1: MPR News
  source_1_image: mdi:newspaper
  source_2: The Current
  source_2_image: mdi:guitar-electric
  source_3: "Shut up, it's fun 🍺"
  source_3_image: mdi:glass-mug-variant
  weather_entity: weather.pirateweather # This is the weather entity it uses to get current temperature
  return_to_default_page: 20s # This is the delay it uses to return to the default page
  encryption_key: !secret encryption_key # Whatever you want to use for the encryption key
  ota_password: !secret ota_password # OTA password to keep folks from ota'ing you
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password
  fallback_password: !secret fallback_password

packages:
  m5dial:
    url: https://gitlab.com/alexives/m5_dial_esphome
    ref: main
    files: 
      # These files are all optional and you can remove them if you want.
      - scale.yaml # Adds the page "ScalePage" that displays automatically when the scale updates
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
        minutes: /1
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
