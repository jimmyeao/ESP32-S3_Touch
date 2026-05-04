# Home Assistant Setup Guide

This guide walks through everything you need to configure on the **Home Assistant** side so the ESPHome panels in this repo work end-to-end.

The panel UI talks to HA over the ESPHome API; every button calls a HA service or reads a HA state. None of the panels store any state of their own (other than the current page and screensaver flag).

---

## 1. Prerequisites

- A working Home Assistant install (Core, OS, or Container — any flavour with the **ESPHome** integration available).
- The **ESPHome** integration enabled (`Settings → Devices & Services → Add Integration → ESPHome`). Once your panel boots, HA will auto-discover it; click **Configure** and paste the API encryption key from your `secrets.yaml`.
- File access to your HA `config/` folder (Samba, SSH, File Editor, or the VS Code add-on) for editing `configuration.yaml` and dropping files into `www/`.

---

## 2. Required entities

The substitutions block at the top of each YAML maps to specific HA entity IDs. You **don't have to use these exact IDs** — just edit the substitutions to match what you have. The panel needs:

| Substitution | Type | Purpose |
|---|---|---|
| `climate_entity` | `climate.*` | Thermostat to read/control |
| `climate_boost_sensor` | `binary_sensor.*` | Reflects "boost active" state (lights up the flame button) |
| `climate_boost_automation` | `automation.*` | Toggled when the flame button is tapped |
| `media_player_entity` | `media_player.*` | Sonos / Spotify / any HA media player |
| `light_1_entity` … `light_4_entity` | `light.*` | Four lights on the Lights page |

If you don't have a thermostat boost setup, you can either:
- Point `climate_boost_sensor` at any `binary_sensor` you don't care about and remove the boost button from the YAML, or
- Create a stub `input_boolean` and a stub automation just so the entities exist.

---

## 3. Background image

Both panels show a custom background image loaded from HA's `/local/` URL (which serves files from `config/www/`).

1. Create `config/www/` if it doesn't exist.
2. Drop a JPEG named `background.jpg` in there, resized to match your display:
   - **CrowPanel 5"** → 800×480
   - **Guition 3.5"** → 320×480
3. Update the `background_image_url` and `background_image_size` substitutions in the YAML if you used a different filename or size.

The image is downloaded once per boot and pushed to every page.

---

## 4. Playlist & Radio browsers (CrowPanel 5" only)

The CrowPanel config has a **dynamic browser** — the 6 playlist slots and 6 radio slots are populated at runtime from HA `input_select` helpers, so you can reorder, rename, or replace entries **without reflashing the panel**.

This requires four pieces in HA: two `input_select` helpers, two template sensors, and two scripts.

### 4a. `input_select` helpers

Add to `configuration.yaml`:

```yaml
input_select:
  crowpanel_playlists:
    name: Crowpanel Playlists
    options:
      - "Favourites"
      - "Chill Out"
      - "80s"
      - "90s"
      - "Coldplay"
      - "Trip Hop"
  crowpanel_radios:
    name: Crowpanel Radios
    options:
      - "BBC Radio 1"
      - "BBC R1 Anthems"
      - "BBC Radio 2"
      - "BBC Radio 4"
```

The panel shows the **first 6 options** of each list. Extras are hidden, so you can keep more than 6 in the helper if you like.

### 4b. Template sensors (join options with `|`)

The panel parses a single pipe-delimited string. Add to `configuration.yaml`:

```yaml
template:
  - sensor:
      - name: "Crowpanel Playlist Names"
        state: "{{ state_attr('input_select.crowpanel_playlists', 'options') | join('|') }}"
      - name: "Crowpanel Radio Names"
        state: "{{ state_attr('input_select.crowpanel_radios', 'options') | join('|') }}"
```

These produce `sensor.crowpanel_playlist_names` and `sensor.crowpanel_radio_names`. The panel subscribes to both; whenever the underlying `input_select` changes, the panel relabels its slots automatically.

### 4c. Scripts (name → URI lookup)

The panel sends only the **friendly name** of the tapped button. Each script holds the `name → URI` mapping and calls `media_player.play_media`.

```yaml
script:
  crowpanel_play_playlist:
    alias: Crowpanel - Play Playlist
    fields:
      name:
        description: Friendly name selected on the panel
    sequence:
      - variables:
          uri_map:
            "Favourites":  "spotify:playlist:1NY1ZsQPf1R8gwBQZx1jIG"
            "Chill Out":   "spotify:playlist:4530EUiHb3H3OVxaOGRIan"
            "80s":         "spotify:playlist:4CJaBASVuHNLT4XJ5l5Y3Y"
            "90s":         "spotify:playlist:37i9dQZF1EQn2GRFTFMl2A"
            "Coldplay":    "spotify:playlist:37i9dQZF1DXaQm3ZVg9Z2X"
            "Trip Hop":    "spotify:playlist:732Y3vA0pCj2In4F5aGXjQ"
      - service: media_player.play_media
        target:
          entity_id: media_player.living_room
        data:
          media_content_id: "{{ uri_map[name] }}"
          media_content_type: "playlist"

  crowpanel_play_radio:
    alias: Crowpanel - Play Radio
    fields:
      name:
        description: Friendly name selected on the panel
    sequence:
      - variables:
          uri_map:
            "BBC Radio 1":     "http://as-hls-ww-live.akamaized.net/pool_01505109/live/ww/bbc_radio_one/bbc_radio_one.isml/bbc_radio_one-audio%3d96000.norewind.m3u8"
            "BBC R1 Anthems":  "http://as-hls-uk-live.akamaized.net/pool_11351741/live/uk/bbc_radio_one_anthems/bbc_radio_one_anthems.isml/bbc_radio_one_anthems-audio%3d96000.norewind.m3u8"
            "BBC Radio 2":     "http://as-hls-ww-live.akamaized.net/pool_74208725/live/ww/bbc_radio_two/bbc_radio_two.isml/bbc_radio_two-audio%3d96000.norewind.m3u8"
            "BBC Radio 4":     "http://as-hls-ww-live.akamaized.net/pool_55057080/live/ww/bbc_radio_fourfm/bbc_radio_fourfm.isml/bbc_radio_fourfm-audio%3d96000.norewind.m3u8"
      - service: media_player.play_media
        target:
          entity_id: media_player.living_room
        data:
          media_content_id: "{{ uri_map[name] }}"
          media_content_type: "music"
```

Replace `media_player.living_room` with your `media_player_entity` (must match the substitution in the YAML).

For Sonos with HLS streams, `media_content_type: "music"` works in most HA/Sonos versions. If a stream fails, try `audio/x-mpegurl` instead.

### 4d. Adding a new entry later

Two edits in HA, no panel reflash:

1. Add the friendly name to the `input_select.options` (the slot label updates instantly).
2. Add `"Friendly Name": "uri:..."` to the matching script's `uri_map` (so taps actually play it).

Reload the helper + scripts from **Developer Tools → YAML** after editing.

---

## 5. Spotify integration

### CrowPanel (standard HA media_player)

The CrowPanel config calls `media_player.play_media` with a Spotify URI directly. This works if you have:

- The official **Spotify** integration installed (`Settings → Devices & Services → Add Integration → Spotify`), **and**
- A Spotify Premium account, **and**
- A target speaker (Sonos, Cast, etc.) — Spotify Connect needs a target device, not the Spotify integration itself.

The cleanest pattern is to point `media_player_entity` at your **speaker** (e.g. `media_player.living_room`) and let HA route the playback. Sonos accepts `spotify:playlist:...` URIs natively.

### Guition (SpotifyPlus)

The Guition config uses the **SpotifyPlus** HACS integration, which exposes richer Spotify control via `spotifyplus.player_media_play_context`. Install it from HACS → Integrations and follow its setup wizard. The Guition YAML's substitutions block contains the SpotifyPlus-specific entity IDs.

---

## 6. Climate boost automation (optional)

The flame button on the Climate page calls `automation.trigger` on whatever you set as `climate_boost_automation`. A minimal example:

```yaml
automation:
  - alias: "Thermostat Boost Toggle"
    id: thermostat_boost_toggle
    trigger: []
    action:
      - service: climate.set_preset_mode
        target:
          entity_id: climate.thermostat_2
        data:
          preset_mode: "boost"
```

Pair it with a `binary_sensor.template` that reflects whether the preset is currently active, and assign that to `climate_boost_sensor`. The button glows orange while boost is on.

---

## 7. Verifying the integration

After flashing the panel and adding the HA config:

1. Reload `configuration.yaml` (Developer Tools → YAML → All YAML configuration) or restart HA.
2. Watch the panel boot — the climate page should show your current temperature within ~5 seconds.
3. Tap **Music** → the album art and track info should match what's playing on `media_player_entity`.
4. Tap **Radio** → 4 BBC stations should be visible (assuming you used the example helper options).
5. Tap a radio station → it should start playing on your speaker within ~2–3 seconds.
6. Wait `screensaver_timeout_s` seconds (default 60) — the screen should switch to the red clock.

If a button does nothing, check:
- `Settings → Devices & Services → ESPHome → <your panel> → Logs` for service-call errors.
- `Developer Tools → Services` to verify the script/automation runs manually.
- The `media_player_entity` is actually playable (Sonos online, Spotify token valid, etc.).

---

## 8. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Panel shows "Loading…" forever for temperature | `climate_entity` doesn't exist or has no `current_temperature` attribute |
| Album art never updates | `entity_picture` from your media player is a relative path AND `ha_base_url` is wrong (missing port, wrong host) |
| Playlist slots are blank | `sensor.crowpanel_playlist_names` doesn't exist or returns "unknown" — check the template sensor in Developer Tools → States |
| Tapping a playlist plays nothing | The friendly name in the `input_select` doesn't match a key in the script's `uri_map` (check spelling/case exactly) |
| Radio plays for a few seconds then stops | Sonos rejected the stream — try `media_content_type: "audio/x-mpegurl"` in the radio script |
| Lights buttons don't toggle | `light_*_entity` is wrong, or the entity isn't a `light.*` (toggling switches needs `switch.toggle` instead) |
