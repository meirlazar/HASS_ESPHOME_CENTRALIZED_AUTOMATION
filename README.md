# HASS Room Presence & Automation Engine

A highly decoupled, state-driven home automation framework for Home Assistant and ESPHome. This repository contains the core logic engine, templating macros, and hardware configurations used to manage complex room states, occupancy triggers, and intelligent routing.

## 🏗️ Architecture Overview

This framework abandons traditional, monolithic Home Assistant automations in favor of a strictly separated **Logic & Execution** model. 

1. **The Logic (`auto.jinja`):** Acts as the central processing engine. It ingests state changes (from ESPHome radar/motion sensors, doors, etc.) and a static rule dictionary (`room_rules`), evaluates the conditions, and outputs a structured JSON payload dictating exactly what should happen.
2. **The Execution (Master Area Control Router):** A single, lean Home Assistant automation that catches triggers, passes them to the Jinja engine, parses the returned JSON payload, and routes the actual service calls (e.g., turning on lights, firing Gotify notifications, updating global state).

### Core Components

* `auto.jinja`: The Jinja2 macro library. Handles edge-detection (tracking `trigger.from_state` to `trigger.to_state`), state evaluation, logging deduplication, and JSON payload construction.
* `room_rules.yaml`: The configuration dictionary defining the expected behaviors, alert thresholds, and lighting states for specific zones (e.g., `parents_room`, `kid1`, `server_closet`).
* `/esphome/`: Hardware definitions for custom ESP32/ESP8266 nodes, including millimeter-wave radar occupancy sensors and custom trigger-based environmental sensors.

## 🚀 Key Features

* **Stateless Macro Processing:** Macros dynamically evaluate edge-triggers without relying on sluggish Home Assistant helpers.
* **Unified Logging (`log_meta`):** Generates clean, human-readable audit trails for every state change. Automatically resolves `entity_id` to `friendly_name` and deduplicates redundant hardware IDs (e.g., switches converting to fans).
* **Smart Alerting:** Integrated Gotify payload generation. The engine determines *if* a notification is needed based on room-specific rule overrides (e.g., `alert_on_entry`) and builds the message before passing it to the router.
* **Global & Local Overrides:** Supports granular action blocking (e.g., `regex_excluded_up`, `do_not_turn_off`) to prevent automations from fighting manual user overrides.

## 🛠️ Configuration Example

**1. Define the Room Rule:**
```yaml
# room_rules configuration
parents_room:
  alert_on_entry: true
  alert_title: "Occupancy Alert"
  alert_msg: "Radar triggered occupancy in the parents room."


# Inside the Master Area Control Router
action:
  - variables:
      old_state: "{{ trigger.from_state.state | default('unknown') }}"
      new_state: "{{ trigger.to_state.state | default('unknown') }}"
      alert_data: >
        {% from 'auto.jinja' import check_new_occupancy, needs_alert %}
        {% set just_occupied = check_new_occupancy(old_state, new_state) %}
        {{ needs_alert(trigger.id, just_occupied, room_rules) | from_json }}
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ alert_data.notify_gotify | default(false) }}"
        sequence:
          - action: notify.gotify
            data:
              title: "{{ alert_data.gotify_title }}"
              message: "{{ alert_data.gotify_message }}"

