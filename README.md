# vtdex_sim_data_collection

This is a meta-repo pinning the set of repositories used to collect trajectories in simulation to train visuotactile dexterous manipulation policies. Each directory is a git submodule tracking an independent upstream repository at a specific commit. To use it, please scroll down to the Rules section, and adhere to the codebase rules. 

| Path | Upstream | Branch | Role |
|---|---|---|---|
| `genesis_adaptor/` | `UniEnvOrg/genesis_adaptor` | `tactile_sensing` | Genesis physics engine → UniEnv adaptor: scenes, robots, sensors, controllers, reference policies |
| `tianji_sim_real/` | `realquantumcookie/tianji_sim_real` | `tactile_sensing` | Sim and real env composers for the Tianji Marvin dual-arm robot + Wuji dexterous hands |
| `unienv/UniEnv/` | `UniEnvOrg/UniEnv` | `main` | Core UniEnv framework (`World`, `WorldNode`, `Env`, spaces, compute backends) |
| `unienv/unienv-input-devices/` | `UniEnvOrg/unienv-input-devices` | `main` | Teleop input devices (AVP, WiLoR, SpaceMouse) |
| `unienv/unienv-tianji/` | `UniEnvOrg/unienv-tianji` | `main` | Tianji arm hardware driver |
| `unienv/unienv-wuji/` | `UniEnvOrg/unienv-wuji` | `main` | Wuji dexterous hand hardware driver |

## Install instructions

```bash
git clone <this-repo-url>
git update --init --recursive submodules
pip install --no-deps -r data_collection_requirements.txt
```

# Pre-data collection checklist

1.  You will need a WiFi connection on the PC that hosts the simulator. Pranav's PC '''spruce-ws''' does not have an inbuilt WiFi adapter. Check if the front of the PC has a USB WiFi adapter: if not, contact Pranav. 

2.  Setup the Apple Vision Pro (AVP): check if battery has enough power, perform eye/hand calibrations as required.

3.  Connect both PC and AVP to the RedRover wifi. 

4.  On the AVP, open the app called 'Tracking Streamer'. If steps 1-3 have been done correctly, you should see the IP address. Note down this IP address (referred to as 'AVP IP' in further steps), and input this into the command you will use for data collection. 

# Data collection instructions

To collect data, run the following command from inside `tianji_sim_real/`:

```bash
python scripts/teleop.py --sim --device avp --ip <AVP IP> --task drill_board \
    --record-dir data/drill_board --sim-cams --camera-depth --hand-tactile --video
```

### What each flag in that command does

| Flag | Meaning |
|---|---|
| `--sim` | Drive the Genesis simulator instead of the real robot. Mutually exclusive with `--real`; one of the two is required. |
| `--device avp` | Take hand/wrist poses from the Apple Vision Pro Tracking Streamer. Other choices are `wilor` (webcam hand tracking) and `wilor-zed`. |
| `--ip <AVP IP>` | Address shown by the Tracking Streamer app on the AVP (step 4 of the checklist). Required for a connected AVP. |
| `--task drill_board` | Load the sim eval task, which spawns the task fixture and emits success/failure. `drill_board` and `hammer_on_shelf` are the registered tasks; omit for a bare tabletop. Sim only. |
| `--record-dir data/drill_board` | Write episodes as `episode_NNNN.pt` under this directory, and arm the keyboard episode controls (see below). Without it nothing is saved. Sim only. |
| `--sim-cams` | Turn on the front/side sim cameras so their images land in the recorded observations. Off by default — teleop itself does not need them, but data collection does. |
| `--camera-depth` | Add a depth channel to the front camera. Requires `--sim-cams`. |
| `--hand-tactile` | Record the full-hand Wuji tactile readings (`obs["hand_tactile"]`). Sim only, and the whole point of the visuotactile dataset. |
| `--video` | Also write `episode_NNNN.mp4` next to each kept episode: front camera + per-hand current/commanded skeleton + tactile panel. Requires `--record-dir`. Useful for eyeballing a demo before trusting it. |

### Keyboard controls while recording

Keys are read from **the terminal that launched the script**, not the viewer window:

| Key | Effect |
|---|---|
| `k` | Keep the current episode (save `.pt`, and `.mp4` if `--video`), then reset the scene. |
| `x` | Discard the current episode and reset the scene. |
| `q` | Quit. |

A task `terminated`/`truncated` (i.e. the task itself reported success or failure) counts as an implicit keep — the episode is saved and the scene resets automatically.

### Other flags worth knowing

Everything below has a working default; change them only if something feels wrong.

| Flag | Default | When to touch it |
|---|---|---|
| `--no-viewer` | viewer on | Run headless on a machine with no display. |
| `--mirror` | off | Bilateral mirror — your left hand drives the robot's right side. Natural when you stand facing the robot. |
| `--scale` | `1.0` | Scale operator hand translation into robot translation. Below 1 for finer control in a small workspace. |
| `--ema` / `--hand-ema` | `0.5` / `0.65` | Smoothing on arm / finger targets. Raise for jittery tracking, at the cost of lag. |
| `--max-lin-vel` / `--max-rot-vel` | `0.25` m/s / `1.0` rad/s | Velocity clamps on the arm targets. |
| `--gesture-fist` / `--gesture-open` / `--gesture-window` | `1.25` / `1.55` / `2.5` s | Thresholds for the fist-then-open re-anchor gesture that engages/detaches an arm. `--no-reanchor-gesture` disables it. |
| `--no-arms` / `--no-hands` | both enabled | Freeze one half of the embodiment while debugging the other. |
| `--dt` | `1/15` s | Control period; also the recorded video fps. |
| `--steps` | `0` (unlimited) | Stop after N steps instead of running until interrupted. |
| `--log-hz` | `1.0` | Rate of the per-side status line (`active`, `engaged`, grip ratio, xyz, mean finger flex). |
| `--offline` | off | Build the env without connecting to any device — smoke-testing the pipeline. |

`--real`-only flags (`--left-hand-serial`, `--right-hand-serial`, `--hand-filter-hz`, `--hand-max-temp`, `--hand-warn-temp`) and the WiLoR capture/camera-axis flags (`--camera-id`, `--width`, `--height`, `--focal-length`, `--camera-x`, `--camera-y`, `--alignment`, `--zed-serial`) do not apply to AVP sim collection; the script rejects them in that combination. Run `python scripts/teleop.py --help` for the full list.

## Working in a submodule

Submodules start in detached HEAD at the pinned commit. To do work, create your own branch both in the meta-repo and in the submodule you are using. 

```bash

git checkout -b <your own meta-repo branch name>
cd genesis_adaptor
git checkout -b <your own repo branch name>     

```
