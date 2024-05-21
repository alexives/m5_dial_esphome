## M5 Dial ESPHome

This is a configuration for managing m5 dials.

To use, you can use the example configuration to set up your dial with various components. For the weather config, you'll also need to set up forecast entities.

Example Config:

```yaml
esphome:
  name: bathroom-dial

substitutions:
  room: Bathroom
  default_page: WeatherPage
  secondary_page: MusicPage
  media_player_control: media_player.bathroom
  scale_entity: sensor.mi_body_composition_scale_61dc_mass
  script_delay: 1s
  weather_entity: weather.brimhall_zoo
  return_to_default_page: 20s
  encryption_key: !secret encryption_key
  ota_password: !secret ota_password
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

display:
  - id: !extend dial_display
    on_page_change:
      - then:
          - delay: $return_to_default_page
          - display.page.show: $default_page

```