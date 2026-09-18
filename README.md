# Motion-Activated Light with Restart-Safe Timer

A Home Assistant automation blueprint that turns a light, switch, or scene on with motion and off after a configurable no-motion delay, using a `timer` helper instead of an in-automation `delay:` or `wait_for_trigger:`.

## Why this exists

Many motion-lighting blueprints (including the one this started from) implement the "turn off after N minutes of no motion" behavior with a `delay:` step sitting inside the automation's action sequence, sometimes preceded by a `wait_for_trigger:` step waiting for motion to clear first.

That pattern has a real failure mode: if Home Assistant restarts while the automation is suspended mid-wait, for an update, a crash, a brief power blip, anything, the pending "turn off" step is lost. There is no recovery. The light or switch can be left on indefinitely with nothing left to turn it off.

This blueprint avoids that entirely by using a `timer` helper to track the countdown instead. A timer helper is its own entity with its own state, and with its restore option enabled, it survives a restart with the correct remaining time. The automation itself never sits suspended waiting for anything; it reacts to three instantaneous triggers (motion on, motion off, timer finished) and returns to idle between each.

## Credit

Forked in spirit from [freakshock88/motion_illuminance_activated_entity.yaml](https://gist.github.com/freakshock88/2311759ba64f929f6affad4c0a67110b), which this blueprint is functionally similar to but does not share code with. The trigger/action structure was rebuilt from scratch around the timer helper.

## Requirements

- A dedicated `timer` helper for each instance of this blueprint. **Do not share one timer across multiple automations built from this blueprint** — each needs its own.
- On the timer helper, enable "Restore state and time when Home Assistant starts?" This is what makes the countdown survive a restart. Without it, you get no benefit over the original delay-based approach.

## Installation

1. Settings > Devices & Services > Helpers > Create Helper > Timer. Enable the restore option. Repeat for each area you plan to use this blueprint in.
2. Save `motion_timer_helper.yaml` to `/config/blueprints/automation/` on your Home Assistant instance (a subfolder such as `/config/blueprints/automation/local/` also works; just make sure the `path:` in any automation using it matches wherever you put it).
3. Reload blueprints (Developer Tools > YAML > Reload Blueprints), or restart Home Assistant.
4. Create a new automation, select this blueprint, and fill in the inputs below.

## Inputs

| Input | Required | Description |
|---|---|---|
| `motion_sensor` | Yes | The motion sensor (or group) that triggers the automation. |
| `target_entity` | Yes | The light, switch, or scene to turn on when motion is detected. |
| `no_motion_timer` | Yes | The timer helper dedicated to this automation instance. |
| `no_motion_wait` | No | An `input_number` holding the number of minutes to wait after motion clears before turning off. If left unset, the target turns off immediately when motion clears. |
| `target_off_entity` | No | If set, this entity is turned off instead of `target_entity`. Useful when the "on" action is a script or scene but there's a specific light/switch entity that actually needs to be turned off. |
| `illuminance_sensor` / `illuminance_cutoff` | No | If both are set, the light only turns on when the illuminance sensor reads below the cutoff value (an `input_number`). |
| `blocker_entity` | No | If this entity's state is `on`, motion will not turn the target on. |
| `turn_off_blocker_entity` | No | If this entity's state is `on`, the target will not be turned off, either when motion clears or when the timer finishes. |
| `time_limit_before` / `time_limit_after` | No | `input_datetime` helpers restricting when the automation is allowed to turn the target on. Supports a window that spans midnight. |

## A known limitation (this is a Home Assistant limitation, not specific to this blueprint)

Per the [official timer documentation](https://www.home-assistant.io/integrations/timer/):

> Automations using the timer.finished event will not trigger on startup if the timer expires when Home Assistant is not running.

In other words: a restart happening *while the timer still has time left* is fully handled, the countdown resumes correctly and the target will still turn off on schedule. A restart happening to span exactly across the moment the timer *would have* finished is not fully handled: the timer entity correctly settles to idle, but the `timer.finished` event that this blueprint listens for does not fire, so the turn-off action never runs.

If this matters for your use case, a simple mitigation is a small separate automation that runs on Home Assistant startup and turns the target off if there's currently no motion and it's still on:

```yaml
alias: <Area> - Startup Safety Net
triggers:
  - trigger: homeassistant
    event: start
conditions:
  - condition: state
    entity_id: <your motion sensor>
    state: "off"
  - condition: state
    entity_id: <your target entity>
    state: "on"
actions:
  - action: homeassistant.turn_off
    target:
      entity_id: <your target entity>
mode: single
```

## License

MIT. Use it, fork it, change it.
