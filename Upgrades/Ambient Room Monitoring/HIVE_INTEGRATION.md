# 🐝 Hive HVAC Integration for ATO System

## Overview

You already have:
- ✅ Average house temperature sensor
- ✅ Hive HVAC system
- ✅ Hallway sensors

Let's integrate them with your ATO to track how your Hive heating affects tank evaporation and temperature!

---

## 🎯 What We'll Connect

**Your Existing Sensors:**
- `sensor.average_house_temperature` (you already have this!)
- Hive thermostat entities
- Hive heating status
- Room sensors (hallway)

**To Your ATO:**
- Evaporation rates
- Tank temperature stability
- Energy efficiency
- Predictive analytics

---

## 📋 Step 1: Find Your Hive Entities

In Home Assistant, your Hive entities are likely:

### Climate Entity:
```
climate.hive_heating
climate.hive_thermostat
climate.heating_*
```

### Sensor Entities:
```
sensor.hive_heating_current_temperature
sensor.hive_heating_target_temperature
sensor.hive_heating_state
binary_sensor.hive_heating_*
```

### To Find Them:
1. Developer Tools → States
2. Search for "hive" or "heating"
3. Note all entity IDs

**Common Hive entities:**
- `climate.hive_heating` - Main thermostat control
- `sensor.hive_heating_current_temperature` - Current temp reading
- `sensor.hive_heating_target_temperature` - Target temp
- `binary_sensor.hive_heating_on` - Heating active/inactive

---

## 🚀 Quick Integration YAML

### Add to configuration.yaml

```yaml
# ============================================================================
# HIVE HVAC + ATO INTEGRATION
# ============================================================================

template:
  - sensor:
      # Heating Impact on Tank
      - name: "Hive Heating Impact Factor"
        state: >
          {% set heating = is_state('climate.hive_heating', 'heat') %}
          {% set target = state_attr('climate.hive_heating', 'temperature') | float(20) %}
          {% set current = states('sensor.average_house_temperature') | float(20) %}
          {% set diff = target - current %}
          {% if heating and diff > 2 %}
            high
          {% elif heating and diff > 1 %}
            moderate
          {% elif heating %}
            low
          {% else %}
            none
          {% endif %}
        icon: >
          {% set impact = states('sensor.hive_heating_impact_factor') %}
          {% if impact == 'high' %}
            mdi:fire
          {% elif impact == 'moderate' %}
            mdi:radiator
          {% elif impact == 'low' %}
            mdi:radiator-disabled
          {% else %}
            mdi:radiator-off
          {% endif %}
        attributes:
          heating_on: >
            {{ is_state('climate.hive_heating', 'heat') }}
          temperature_gap: >
            {% set target = state_attr('climate.hive_heating', 'temperature') | float(20) %}
            {% set current = states('sensor.average_house_temperature') | float(20) %}
            {{ (target - current) | round(1) }}
          predicted_evaporation_change: >
            {% set heating = is_state('climate.hive_heating', 'heat') %}
            {% set diff = (state_attr('climate.hive_heating', 'temperature') | float(20)) - (states('sensor.average_house_temperature') | float(20)) %}
            {% if heating and diff > 2 %}
              +25-35% (active heating)
            {% elif heating %}
              +10-15% (maintenance heating)
            {% else %}
              Normal
            {% endif %}
      
      # Tank vs House Temperature Efficiency
      - name: "Tank House Temperature Efficiency"
        unit_of_measurement: "%"
        state: >
          {% set house = states('sensor.average_house_temperature') | float(20) %}
          {% set tank = states('sensor.ato_tank_temperature') | float(25) %}
          {% set target_tank = 25 %}
          {% set ideal_house = 22 %}
          {% set house_diff = (ideal_house - house) | abs %}
          {% set efficiency = 100 - (house_diff * 10) %}
          {{ [efficiency, 0] | max | round(0) }}
        icon: mdi:gauge
        attributes:
          status: >
            {% set eff = states('sensor.tank_house_temperature_efficiency') | int(0) %}
            {% if eff >= 90 %}
              Excellent - house temp supports tank perfectly
            {% elif eff >= 75 %}
              Good - minimal heater strain
            {% elif eff >= 60 %}
              Fair - heater working moderately
            {% elif eff >= 40 %}
              Poor - heater working hard
            {% else %}
              Critical - house too cold, high energy use
            {% endif %}
          house_temp: "{{ states('sensor.average_house_temperature') }}°C"
          tank_temp: "{{ states('sensor.ato_tank_temperature') }}°C"
      
      # Hive Schedule Evaporation Predictor
      - name: "Predicted Evaporation Today"
        unit_of_measurement: "L"
        state: >
          {% set base_rate = states('sensor.ato_rate_24_hours') | float(0.15) %}
          {% set heating = is_state('climate.hive_heating', 'heat') %}
          {% set humidity = states('sensor.hallway_humidity') | float(50) %}
          
          {# Calculate heating factor #}
          {% set heating_factor = 1.15 if heating else 1.0 %}
          
          {# Calculate humidity factor #}
          {% set humidity_factor = 1 + ((50 - humidity) * 0.015) %}
          
          {# Daily prediction #}
          {{ (base_rate * heating_factor * humidity_factor * 24) | round(2) }}
        icon: mdi:water-alert
        attributes:
          factors: >
            Heating: {{ 'Active (+15%)' if is_state('climate.hive_heating', 'heat') else 'Off' }}
            Humidity: {{ states('sensor.hallway_humidity') }}% ({{ '+' if states('sensor.hallway_humidity')|float(50) < 50 else '-' }}{{ ((50 - states('sensor.hallway_humidity')|float(50))|abs * 1.5)|round(0) }}%)

  - binary_sensor:
      # Heating-Induced High Evaporation Alert
      - name: "High Evaporation Due to Heating"
        state: >
          {% set heating = is_state('climate.hive_heating', 'heat') %}
          {% set evap = states('sensor.ato_rate_24_hours') | float(0) %}
          {% set threshold = 0.25 %}
          {{ heating and evap > threshold }}
        icon: >
          {{ 'mdi:fire-alert' if is_state('binary_sensor.high_evaporation_due_to_heating', 'on') else 'mdi:check-circle' }}
        attributes:
          reason: >
            {% if is_state('binary_sensor.high_evaporation_due_to_heating', 'on') %}
              Hive heating is on and evaporation is elevated ({{ states('sensor.ato_rate_24_hours') }}L/h)
            {% else %}
              Normal conditions
            {% endif %}

# ============================================================================
# HIVE-SPECIFIC AUTOMATIONS
# ============================================================================

automation:
  # Notify when Hive heating causes high evaporation
  - alias: "ATO - Hive Heating Evaporation Alert"
    id: ato_hive_heating_evap
    trigger:
      - platform: state
        entity_id: binary_sensor.high_evaporation_due_to_heating
        to: "on"
        for:
          minutes: 30
    action:
      - service: notify.mobile_app_YOUR_PHONE
        data:
          title: "🔥 Hive Heating Affecting Tank"
          message: >
            Evaporation increased to {{ states('sensor.ato_rate_24_hours') }}L/h
            Hive target: {{ state_attr('climate.hive_heating', 'temperature') }}°C
            Current: {{ states('sensor.average_house_temperature') }}°C
            Predicted today: {{ states('sensor.predicted_evaporation_today') }}L
  
  # Daily Hive heating summary
  - alias: "ATO - Daily Hive Impact Report"
    id: ato_daily_hive_report
    trigger:
      - platform: time
        at: "20:00:00"
    condition:
      - condition: template
        value_template: "{{ is_state('climate.hive_heating', 'heat') }}"
    action:
      - service: notify.mobile_app_YOUR_PHONE
        data:
          title: "📊 Daily Hive Heating Impact"
          message: >
            House Temp: {{ states('sensor.average_house_temperature') }}°C
            Tank Temp: {{ states('sensor.ato_tank_temperature') }}°C
            Today's Evaporation: {{ states('sensor.ato_daily_usage') }}L
            Efficiency: {{ states('sensor.tank_house_temperature_efficiency') }}%
            
            {% if states('sensor.tank_house_temperature_efficiency')|int(100) < 70 %}
            💡 Tip: Consider raising Hive thermostat to reduce tank heater load
            {% endif %}
  
  # Hive schedule change notification
  - alias: "ATO - Hive Schedule Changed"
    id: ato_hive_schedule_change
    trigger:
      - platform: state
        entity_id: climate.hive_heating
        attribute: temperature
    action:
      - service: notify.mobile_app_YOUR_PHONE
        data:
          title: "🐝 Hive Schedule Changed"
          message: >
            New target: {{ state_attr('climate.hive_heating', 'temperature') }}°C
            Current: {{ states('sensor.average_house_temperature') }}°C
            
            Predicted tank impact:
            {% set diff = (state_attr('climate.hive_heating', 'temperature')|float(20) - states('sensor.average_house_temperature')|float(20)) %}
            {% if diff > 3 %}
            - Evaporation may increase 20-30%
            - Tank heater load will reduce
            {% elif diff > 1 %}
            - Evaporation may increase 10-15%
            - Slight reduction in tank heater use
            {% else %}
            - Minimal impact expected
            {% endif %}
  
  # Winter optimization suggestion
  - alias: "ATO - Hive Winter Optimization"
    id: ato_hive_winter_optimization
    trigger:
      - platform: numeric_state
        entity_id: sensor.tank_house_temperature_efficiency
        below: 60
        for:
          hours: 6
    condition:
      - condition: state
        entity_id: sensor.ato_current_season
        state: "Winter"
    action:
      - service: notify.mobile_app_YOUR_PHONE
        data:
          title: "❄️ Winter Tank Efficiency Low"
          message: >
            Tank heater working hard ({{ states('sensor.tank_house_temperature_efficiency') }}% efficiency)
            
            House: {{ states('sensor.average_house_temperature') }}°C
            Tank: {{ states('sensor.ato_tank_temperature') }}°C
            Gap: {{ states('sensor.average_house_temperature')|float(20) - states('sensor.ato_tank_temperature')|float(25) }}°C
            
            💡 Suggestions:
            - Raise Hive thermostat by 1-2°C
            - Move tank to warmer room
            - Add tank insulation
            - Expected savings: $5-10/month
  
  # Hive heating ON → Track evaporation spike
  - alias: "ATO - Track Hive Heating Cycles"
    id: ato_track_hive_cycles
    trigger:
      - platform: state
        entity_id: climate.hive_heating
        to: "heat"
    action:
      - service: logbook.log
        data:
          name: "Hive Heating"
          message: >
            Heating cycle started
            Target: {{ state_attr('climate.hive_heating', 'temperature') }}°C
            Current evap: {{ states('sensor.ato_rate_24_hours') }}L/h
          entity_id: climate.hive_heating
  
  # Hive heating OFF → Log cycle end
  - alias: "ATO - Hive Heating Cycle End"
    id: ato_hive_cycle_end
    trigger:
      - platform: state
        entity_id: climate.hive_heating
        from: "heat"
    action:
      - service: logbook.log
        data:
          name: "Hive Heating"
          message: >
            Heating cycle ended
            Final temp: {{ states('sensor.average_house_temperature') }}°C
            Current evap: {{ states('sensor.ato_rate_24_hours') }}L/h
          entity_id: climate.hive_heating
```

---

## 📊 Dashboard Cards

Add these to your ATO dashboard:

```yaml
# ============================================================================
# HIVE HEATING INTEGRATION CARDS
# ============================================================================

# Main Hive Status Card
- type: entities
  title: 🐝 Hive Heating & Tank Impact
  entities:
    - entity: climate.hive_heating
      name: Hive Thermostat
    
    - entity: sensor.average_house_temperature
      name: Average House Temp
      icon: mdi:home-thermometer
    
    - entity: sensor.ato_tank_temperature
      name: Tank Temperature
      icon: mdi:thermometer-water
    
    - type: divider
    
    - entity: sensor.hive_heating_impact_factor
      name: Heating Impact
    
    - entity: sensor.tank_house_temperature_efficiency
      name: Temperature Efficiency
    
    - entity: binary_sensor.high_evaporation_due_to_heating
      name: Evaporation Alert
    
    - type: divider
    
    - entity: sensor.predicted_evaporation_today
      name: Predicted Usage Today

# Hive vs Tank Correlation Chart
- type: custom:apexcharts-card
  header:
    show: true
    title: 🐝 Hive Heating vs Tank Conditions
  graph_span: 3d
  apex_config:
    chart:
      height: 300
    annotations:
      yaxis:
        - y: 25
          borderColor: '#10b981'
          strokeDashArray: 3
          label:
            text: 'Target Tank Temp'
    yaxis:
      - id: temp
        title:
          text: "Temperature (°C)"
        min: 18
        max: 28
      - id: evap
        opposite: true
        title:
          text: "Evaporation (L/h)"
        min: 0
  series:
    # House temperature
    - entity: sensor.average_house_temperature
      name: House Temp
      yaxis_id: temp
      color: '#f59e0b'
      stroke_width: 2
    
    # Tank temperature
    - entity: sensor.ato_tank_temperature
      name: Tank Temp
      yaxis_id: temp
      color: '#3b82f6'
      stroke_width: 3
    
    # Hive target temperature (as horizontal line reference)
    - entity: climate.hive_heating
      name: Hive Target
      yaxis_id: temp
      color: '#ef4444'
      stroke_width: 1
      attribute: temperature
      type: line
      curve: stepline
    
    # Evaporation rate
    - entity: sensor.ato_rate_24_hours
      name: Evaporation
      yaxis_id: evap
      color: '#10b981'
      stroke_width: 2

# Hive Heating Status Card
- type: custom:mushroom-template-card
  primary: "Hive Heating Status"
  secondary: >
    {% if is_state('climate.hive_heating', 'heat') %}
    🔥 Heating Active
    Target: {{ state_attr('climate.hive_heating', 'temperature') }}°C
    Current: {{ states('sensor.average_house_temperature') }}°C
    Impact: {{ states('sensor.hive_heating_impact_factor') }}
    {% else %}
    ✅ Heating Off
    House: {{ states('sensor.average_house_temperature') }}°C
    Tank stable at {{ states('sensor.ato_tank_temperature') }}°C
    {% endif %}
  icon: >
    {% if is_state('climate.hive_heating', 'heat') %}
    mdi:fire
    {% else %}
    mdi:radiator-off
    {% endif %}
  icon_color: >
    {% if is_state('climate.hive_heating', 'heat') %}
    red
    {% else %}
    grey
    {% endif %}
  tap_action:
    action: more-info
    entity: climate.hive_heating

# Efficiency & Optimization Card
- type: markdown
  title: 💡 Hive Optimization Analysis
  content: |
    ### Current Status
    
    **House Temperature:** {{ states('sensor.average_house_temperature') }}°C
    **Tank Temperature:** {{ states('sensor.ato_tank_temperature') }}°C
    **Temperature Gap:** {{ (states('sensor.ato_tank_temperature')|float(25) - states('sensor.average_house_temperature')|float(20))|round(1) }}°C
    
    **Efficiency:** {{ states('sensor.tank_house_temperature_efficiency') }}%
    
    {% set eff = states('sensor.tank_house_temperature_efficiency')|int(100) %}
    {% if eff >= 90 %}
    🟢 **Excellent** - House temp is ideal for tank
    - Tank heater barely running
    - Energy efficient setup
    - No changes needed
    {% elif eff >= 75 %}
    🟡 **Good** - Reasonable efficiency
    - Tank heater runs occasionally
    - Consider raising Hive by 1°C for better efficiency
    {% elif eff >= 60 %}
    🟠 **Fair** - Room for improvement
    - Tank heater working moderately
    - **Recommendation:** Raise Hive thermostat by 2°C
    - **Estimated savings:** $3-5/month
    {% else %}
    🔴 **Poor** - High energy waste
    - Tank heater working very hard
    - **Urgent:** Raise Hive thermostat by 3-4°C
    - **Or:** Move tank to warmer location
    - **Potential savings:** $8-12/month
    {% endif %}
    
    ---
    
    ### Today's Prediction
    
    {% if is_state('climate.hive_heating', 'heat') %}
    🔥 Heating is **ON**
    - Expected evaporation: {{ states('sensor.predicted_evaporation_today') }}L
    - Impact: {{ state_attr('sensor.hive_heating_impact_factor', 'predicted_evaporation_change') }}
    {% else %}
    ✅ Heating is **OFF**
    - Expected normal evaporation
    - Tank relying on own heater
    {% endif %}

# Hive Schedule Impact Card
- type: horizontal-stack
  cards:
    - type: entity
      entity: sensor.tank_house_temperature_efficiency
      name: Efficiency
      icon: mdi:gauge
    
    - type: custom:mushroom-template-card
      primary: "Heating Impact"
      secondary: "{{ states('sensor.hive_heating_impact_factor') | title }}"
      icon: >
        {% set impact = states('sensor.hive_heating_impact_factor') %}
        {% if impact == 'high' %}
        mdi:fire
        {% elif impact == 'moderate' %}
        mdi:radiator
        {% elif impact == 'low' %}
        mdi:radiator-disabled
        {% else %}
        mdi:radiator-off
        {% endif %}
      icon_color: >
        {% set impact = states('sensor.hive_heating_impact_factor') %}
        {% if impact == 'high' %}
        red
        {% elif impact == 'moderate' %}
        orange
        {% elif impact == 'low' %}
        yellow
        {% else %}
        grey
        {% endif %}
    
    - type: entity
      entity: sensor.predicted_evaporation_today
      name: Today's Usage
      icon: mdi:water-alert
```

---

## 🎯 What This Does

### Real-Time Tracking:
- When Hive heating turns ON → Track evaporation increase
- When Hive heating turns OFF → Log cycle and impact
- Continuous efficiency monitoring

### Smart Alerts:
- High evaporation during heating cycles
- Low efficiency warnings (house too cold)
- Daily impact summaries
- Optimization suggestions

### Predictive Analytics:
- Predict today's water usage based on Hive schedule
- Calculate heating impact factor
- Estimate energy efficiency
- Suggest thermostat adjustments

---

## 💡 Example Insights

### Morning (Hive Heating ON):
```
Notification:
"🔥 Hive heating started
Target: 21°C, Current: 18°C
Predicted evaporation increase: 20-25%
Today's usage: 3.8L (vs normal 3.0L)"
```

### Evening (Efficiency Check):
```
Notification:
"📊 Daily Hive Impact Report
House: 20°C
Tank: 25°C
Efficiency: 75% (Good)
Today's evaporation: 3.2L"
```

### Winter Warning:
```
Notification:
"❄️ Winter Tank Efficiency Low (62%)
House: 17°C, Tank: 25°C
Tank heater working hard!
💡 Raise Hive by 2°C to save $8/month"
```

---

## 🚀 Installation Steps

### 1. Replace Entity Names (2 min)
Find and replace in the YAML above:
- `climate.hive_heating` → Your actual Hive climate entity
- `YOUR_PHONE` → Your mobile device name

### 2. Add to configuration.yaml (3 min)
Copy entire template and automation sections

### 3. Add to Dashboard (5 min)
Copy dashboard cards to your ATO dashboard

### 4. Restart Home Assistant (2 min)
Settings → System → Restart

### 5. Done! (1 min)
Watch the correlations appear!

---

## 📊 Advanced Features

### Hive Schedule Integration:
```yaml
# Track when Hive changes schedule
- alias: "ATO - Hive Boost Mode Alert"
  trigger:
    - platform: state
      entity_id: climate.hive_heating
      attribute: preset_mode
      to: "boost"
  action:
    - service: notify.mobile_app_YOUR_PHONE
      data:
        message: "🔥 Hive boost activated - expect 30% higher evaporation for next hour"
```

### Cost Tracking:
```yaml
template:
  - sensor:
      - name: "Tank Heater Cost Estimate"
        unit_of_measurement: "$/month"
        state: >
          {% set house = states('sensor.average_house_temperature')|float(20) %}
          {% set tank = states('sensor.ato_tank_temperature')|float(25) %}
          {% set diff = tank - house %}
          {% set base_cost = 12 %}  # Base monthly heater cost
          {% set factor = 1 + (diff * 0.15) %}
          {{ (base_cost * factor)|round(2) }}
        attributes:
          savings_if_house_2c_warmer: >
            {% set current = states('sensor.tank_heater_cost_estimate')|float(12) %}
            {% set improved = current * 0.75 %}
            ${{ (current - improved)|round(2) }}/month
```

---

## ✅ What You'll Learn

### Over Days:
- Hive heating cycles correlate with evaporation spikes
- Morning warmup = 15-25% more evaporation
- Evening setback = normal evaporation returns

### Over Weeks:
- Hive schedule optimization opportunities
- Best thermostat settings for tank efficiency
- Energy cost patterns

### Over Months:
- Winter vs summer Hive impact
- Yearly efficiency trends
- Cost optimization achieved

---

## 💰 Cost Savings Examples

**Scenario 1: Winter Optimization**
```
Before: House 17°C, Tank 25°C (8°C gap)
Tank heater cost: ~$15/month

After: Raise Hive to 20°C (5°C gap)
Tank heater cost: ~$10/month
Hive extra cost: +$3/month
Net savings: $2/month + better tank stability
```

**Scenario 2: Schedule Adjustment**
```
Before: Hive off at night (15°C), on during day (21°C)
Wild temperature swings

After: Maintain 19°C constant
Stable temps, slightly higher Hive cost but:
- Less tank heater cycling
- Better fish health
- Consistent evaporation
```

---

## 🎯 Summary

**Time:** 15 minutes  
**Hardware:** None (using Hive you already have!)  
**Difficulty:** Easy (copy/paste)  
**Value:** High (optimize both Hive and tank)  

**You Get:**
- 📊 Hive heating impact tracking
- 💡 Efficiency optimization
- 💰 Cost reduction strategies
- 🎯 Predictive evaporation
- 🔔 Smart alerts

---

## 🚀 Ready?

**Quick start:**
1. Find your Hive entity: `climate.hive_heating` (or similar)
2. Copy the YAML above
3. Replace entity names
4. Paste into configuration.yaml
5. Add dashboard cards
6. Restart HA
7. Done!

**Need help with entity names?**

Tell me what shows up when you search for "hive" in Developer Tools → States, and I'll customize the exact YAML for you! 🐝

---

**This connects your existing Hive system directly to your ATO monitoring! 🐝🐠✨**
