### Executive Summary

**The Problem:** Your audio setup on Arch Linux with a KDE/Wayland desktop is unpredictable. Physical devices like your Tascam Model 12 and various synths do not reliably select their correct multi-channel profiles, and their connections (both audio and MIDI) are not persistent. Manually routing with `qpwgraph` is a temporary fix that is lost on reboot or device reconnection. You need a stable, automated system for pro audio work.

**The Solution:** The solution is to move from a reactive, manual approach to a **declarative, automated** one. By creating a set of custom configuration files for PipeWire and its session manager, WirePlumber, you can define exactly how your devices and applications should behave. This configuration will be persistent and applied automatically whenever a device is connected or an application is launched.

**The Core Techniques:**
1.  **Persistent Virtual Sinks:** Creating stable virtual audio devices that act as predictable "junction boxes" for routing audio.
2.  **WirePlumber Profile Rules:** Forcing specific hardware (Tascam, synths) to always select their "Pro Audio" (multi-channel) profile upon connection.
3.  **WirePlumber Linking Rules:** Automatically creating audio and MIDI connections between hardware and software (e.g., MiniBrute -> Ardour, VCV Rack -> Ardour) as soon as they become available.

---

### Validation and Final Evaluation

After reviewing the entire plan, the proposed solutions are robust and address all the issues you've raised. They use the intended, modern PipeWire/WirePlumber mechanisms.

**Points of Refinement & Final Logic:**

1.  **Consolidated Rules:** The rules are logically separated into files based on function (device profiles, MIDI routing, audio routing). The numbered prefixes (`51-`, `52-`, `60-`) ensure a predictable loading order for human organization, though WirePlumber is smart enough to apply them correctly regardless.
2.  **Specificity is Key:** The rules rely on matching names (`device.description`, `node.name`, `application.name`). The key to success is finding the *exact* names using tools like `wpctl status` and `pw-top`. Using wildcards (`*`) is good practice for robustness, but be specific enough to avoid ambiguity (e.g., use `*SP-404MKII*` instead of just `*Roland*`).
3.  **The `module-combine-sink` is a powerful tool:** For your goal of recording and monitoring Ardour's output simultaneously, the `combine-sink` is the ideal solution. You will set Ardour's main output to this sink, and it will automatically feed both your Tascam and your local headphones.
4.  **`apply_properties` vs. `make_links`:** The VCV Rack rules correctly use both methods. `apply_properties` sets a *default* target (your headphones), while `make_links` adds a *secondary* link (to Ardour for recording) without overriding the default. This is the correct pattern for multi-destination routing.
5.  **Troubleshooting is Essential:** If something doesn't work, the cause is almost always a typo or incorrect name in a configuration file. The `journalctl --user -fu wireplumber.service` command is your single most important debugging tool, as it will print errors from your Lua scripts.

The plan is sound. Proceeding with the following steps will create the stable audio environment you need.

---
### Final Step-by-Step Implementation Plan

This is the complete guide. Follow these steps in order.

**Step 0: Pre-flight Check**
Ensure you have a command-line text editor installed, such as `nano` or `vim`. These instructions will use `nano`.

**Step 1: Create Configuration Directories**
Open a terminal and create the necessary folder structure. The `-p` flag creates parent directories as needed.
```bash
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d/
mkdir -p ~/.config/wireplumber/main.lua.d/
```

**Step 2: Create Persistent Virtual & Combined Sinks**
This file will define your stable "junction boxes."

1.  Create the config file:
    ```bash
    nano ~/.config/pipewire/pipewire-pulse.conf.d/10-pro-audio-sinks.conf
    ```
2.  Paste in the following content. **You must replace the example `slaves` with your real device names**, which you can find with `pactl list sinks`.

    ```json
    # In ~/.config/pipewire/pipewire-pulse.conf.d/10-pro-audio-sinks.conf
    context.modules = [
        {
            name = "libpipewire-module-combine-sink",
            args = {
                sink_name = "Record-and-Monitor-Sink",
                sink_properties = {
                    device.description = "Ardour Rec+Monitor (Tascam+Headphones)"
                },
                # !! IMPORTANT: Replace these with your actual device names from `pactl list sinks` !!
                slaves = [
                    "alsa_output.usb-TEAC_CORPORATION_TASCAM_Model_12-00.pro-output-0",
                    "alsa_output.pci-0000_0b_00.4.analog-stereo"
                ],
                grid.mode = "exactly",
                audio.format = "S32LE",
                audio.rate = 48000
            }
        }
    ]
    ```
3.  Save the file and exit (`Ctrl+X`, then `Y`, then `Enter` in `nano`).

**Step 3: Force Devices into "Pro Audio" Mode**
This ensures your hardware always exposes all its channels.

1.  Create a WirePlumber script for your Tascam:
    ```bash
    nano ~/.config/wireplumber/main.lua.d/51-tascam-pro-profile.lua
    ```
2.  Paste in the rule. **Customize the `device.description` and `device.profile`** after finding them with `wpctl status`.
    ```lua
    -- In ~/.config/wireplumber/main.lua.d/51-tascam-pro-profile.lua
    table.insert(alsa_monitor.rules, {
      matches = { { { "device.description", "matches", "*TASCAM*Model*12*" } } },
      apply_properties = { ["device.profile"] = "Pro-Audio-12-in-10-out" },
    })
    ```
3.  Create another script for your other synths:
    ```bash
    nano ~/.config/wireplumber/main.lua.d/52-synth-profiles.lua
    ```
4.  Paste in rules for each synth. **You must find and customize the names for each one.**
    ```lua
    -- In ~/.config/wireplumber/main.lua.d/52-synth-profiles.lua
    -- !! Customize the name and profile for EACH device !!

    -- Rule for the Roland SP-404MKII
    table.insert(alsa_monitor.rules, {
      matches = { { { "device.description", "matches", "*SP-404MKII*" } } },
      apply_properties = { ["device.profile"] = "Pro" },
    })

    -- Rule for the Roland D-05
    table.insert(alsa_monitor.rules, {
      matches = { { { "device.description", "matches", "*D-05*" } } },
      apply_properties = { ["device.profile"] = "Pro" },
    })

    -- Add more rule blocks here for your JU-06A, JX-03, etc.
    ```

**Step 4: Automate Audio & MIDI Routing**
This section creates your automatic patchbay.

1.  Create a script for MIDI routing:
    ```bash
    nano ~/.config/wireplumber/main.lua.d/60-midi-routing.lua
    ```
2.  Add rules to link your MIDI controllers to Ardour. **Customize the `node.name` for your devices.**
    ```lua
    -- In ~/.config/wireplumber/main.lua.d/60-midi-routing.lua
    
    -- Arturia MiniBrute 2s -> Ardour
    table.insert(linking.rules, {
      matches = { { { "node.name", "matches", "*minibrute*2s*:midi/playback*" } } },
      make_links = { { "node.name", "matches", "ardour:Midi/audio_in*" } }
    })
    
    -- Arturia DrumBrute Impact -> Ardour
    table.insert(linking.rules, {
      matches = { { { "node.name", "matches", "*drumbrute*impact*:midi/playback*" } } },
      make_links = { { "node.name", "matches", "ardour:Midi/audio_in*" } }
    })
    ```
3.  Create a script for advanced audio routing (the VCV Rack example):
    ```bash
    nano ~/.config/wireplumber/main.lua.d/61-audio-routing.lua
    ```
4.  Add the dual-routing rule. **Customize the names if needed.**
    ```lua
    -- In ~/.config/wireplumber/main.lua.d/61-audio-routing.lua
    
    table.insert(linking.rules, {
        -- Rule 1: Set DEFAULT output for VCV Rack to your local headphones
        {
            matches = { {
                { "application.name", "matches", "VCV Rack 2 Pro" },
                { "media.class", "matches", "Audio/Source" },
            } },
            apply_properties = {
                -- !! Replace with your actual headphone/speaker sink name !!
                ["node.target"] = "alsa_output.pci-0000_0b_00.4.analog-stereo",
            },
        },
        -- Rule 2: Create a SECONDARY link to Ardour for recording
        {
            matches = { {
                { "application.name", "matches", "VCV Rack 2 Pro" },
                { "media.class", "matches", "Audio/Source" },
            } },
            make_links = {
                { "node.name", "matches", "ardour:audio_in*" },
            }
        }
    })
    ```

**Step 5: Activate and Test**
Restart the services to apply all your new configurations.
```bash
systemctl --user restart pipewire.service pipewire-pulse.service wireplumber.service
```

**Step 6: Verify Your New Workflow**
1.  Plug in your Tascam and synths. Use `wpctl status` to confirm they are in their "Pro" profile without you doing anything.
2.  Launch Ardour. Set its Master Bus output to your new "**Record-and-Monitor-Sink**".
3.  Launch VCV Rack. You should hear it through your headphones. Open `qpwgraph` and you will see it is *also* connected to Ardour's inputs automatically.
4.  Connect your MIDI controllers. Check `qpwgraph` to see they are automatically linked to Ardour's MIDI input.

---

### Complete Configuration as a Markdown File

````markdown
# PipeWire Pro Audio Automation Plan

This document outlines a complete, declarative configuration for a stable pro audio setup using PipeWire and WirePlumber.

## 1. Executive Summary

The goal is to eliminate manual patching and unreliable device profiles by creating a set of persistent, automated rules. This plan forces hardware into multi-channel modes and automatically routes audio and MIDI between specified applications and devices.

## 2. Configuration Files

### File: `~/.config/pipewire/pipewire-pulse.conf.d/10-pro-audio-sinks.conf`

This file creates a persistent "combined" sink, which allows an application (like Ardour) to output to both the Tascam interface and local headphones simultaneously for recording and monitoring.

```json
# In ~/.config/pipewire/pipewire-pulse.conf.d/10-pro-audio-sinks.conf
context.modules = [
    {
        name = "libpipewire-module-combine-sink",
        args = {
            sink_name = "Record-and-Monitor-Sink",
            sink_properties = {
                device.description = "Ardour Rec+Monitor (Tascam+Headphones)"
            },
            # !! IMPORTANT: Replace these with your actual device names from `pactl list sinks` !!
            slaves = [
                "alsa_output.usb-TEAC_CORPORATION_TASCAM_Model_12-00.pro-output-0",
                "alsa_output.pci-0000_0b_00.4.analog-stereo"
            ],
            grid.mode = "exactly",
            audio.format = "S32LE",
            audio.rate = 48000
        }
    }
]
```

### File: `~/.config/wireplumber/main.lua.d/51-tascam-pro-profile.lua`

This WirePlumber rule forces the Tascam Model 12 into its multi-channel profile as soon as it's connected.

```lua
-- In ~/.config/wireplumber/main.lua.d/51-tascam-pro-profile.lua
table.insert(alsa_monitor.rules, {
  matches = { { { "device.description", "matches", "*TASCAM*Model*12*" } } },
  apply_properties = { ["device.profile"] = "Pro-Audio-12-in-10-out" },
})
```

### File: `~/.config/wireplumber/main.lua.d/52-synth-profiles.lua`

This file contains rules to force other USB synths into their correct "Pro" or multi-channel modes.

```lua
-- In ~/.config/wireplumber/main.lua.d/52-synth-profiles.lua
-- !! Customize the name and profile for EACH device using `wpctl status` !!

-- Rule for the Roland SP-404MKII
table.insert(alsa_monitor.rules, {
  matches = { { { "device.description", "matches", "*SP-404MKII*" } } },
  apply_properties = { ["device.profile"] = "Pro" },
})

-- Rule for the Roland D-05
table.insert(alsa_monitor.rules, {
  matches = { { { "device.description", "matches", "*D-05*" } } },
  apply_properties = { ["device.profile"] = "Pro" },
})

-- Add more rule blocks here for your JU-06A, JX-03, etc.
```

### File: `~/.config/wireplumber/main.lua.d/60-midi-routing.lua`

This script automatically connects MIDI controllers to Ardour's MIDI input.

```lua
-- In ~/.config/wireplumber/main.lua.d/60-midi-routing.lua

-- Arturia MiniBrute 2s -> Ardour
table.insert(linking.rules, {
  matches = { { { "node.name", "matches", "*minibrute*2s*:midi/playback*" } } },
  make_links = { { "node.name", "matches", "ardour:Midi/audio_in*" } }
})

-- Arturia DrumBrute Impact -> Ardour
table.insert(linking.rules, {
  matches = { { { "node.name", "matches", "*drumbrute*impact*:midi/playback*" } } },
  make_links = { { "node.name", "matches", "ardour:Midi/audio_in*" } }
})
```

### File: `~/.config/wireplumber/main.lua.d/61-audio-routing.lua`

This script provides advanced routing for applications like VCV Rack, sending its audio to both the default output and to Ardour simultaneously.

```lua
-- In ~/.config/wireplumber/main.lua.d/61-audio-routing.lua

table.insert(linking.rules, {
    -- Rule 1: Set DEFAULT output for VCV Rack to local headphones
    {
        matches = { {
            { "application.name", "matches", "VCV Rack 2 Pro" },
            { "media.class", "matches", "Audio/Source" },
        } },
        apply_properties = {
            -- !! Replace with your actual headphone/speaker sink name !!
            ["node.target"] = "alsa_output.pci-0000_0b_00.4.analog-stereo",
        },
    },
    -- Rule 2: Create a SECONDARY link to Ardour for recording
    {
        matches = { {
            { "application.name", "matches", "VCV Rack 2 Pro" },
            { "media.class", "matches", "Audio/Source" },
        } },
        make_links = {
            { "node.name", "matches", "ardour:audio_in*" },
        }
    }
})
```

## 3. Activation Command

After all files are created and customized, run this command to apply them:

```bash
systemctl --user restart pipewire.service pipewire-pulse.service wireplumber.service
```

## 4. Troubleshooting

If something doesn't work, use these tools to find the issue:
-   **Check for errors in your rules:** `journalctl --user -fu wireplumber.service`
-   **Find correct device/app names:** `wpctl status`, `pw-top`, and `qpwgraph`.

The most common errors are typos in the Lua scripts or incorrect device/application names.
````
