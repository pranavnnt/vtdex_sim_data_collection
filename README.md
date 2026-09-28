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

1.  You will need a WiFi connection on the PC that hosts the simulator. Pranav's PC '''spruce-ws''' does not have an inbuilt WiFi adapter, but now has an unshared WiFi adaptor. 

2.  Setup the Apple Vision Pro (AVP): check if battery has enough power. I highly recommend performing eye/hand calibration at the start of each session as it highly improves the user gaze interface. 

3.  Connect both PC and AVP to the RedRover wifi. 

4.  On the AVP, open the app called 'Tracking Streamer'. If steps 1-3 have been done correctly, you should see the IP address. Note down this IP address (referred to as 'AVP IP' in further steps), and input this into the command you will use for data collection. 

# Data collection instructions

To collect data, run the following command from inside `tianji_sim_real/`:

```bash
python scripts/teleop.py --sim --device avp --ip <AVP IP> --task drill_board \
    --record-dir data/drill_board --sim-cams --camera-depth --hand-tactile
```

For the `grasp_drill` task, follow the step-by-step guide in [docs/grasp_drill_data_collection.md](docs/grasp_drill_data_collection.md).

### What each flag in that command does

| Flag | Meaning |
|---|---|
| `--sim` | Drive the Genesis simulator instead of the real robot. Mutually exclusive with `--real`; one of the two is required. |
| `--device avp` | Take hand/wrist poses from the Apple Vision Pro Tracking Streamer. Other choices are `wilor` (webcam hand tracking) and `wilor-zed`. |
| `--ip <AVP IP>` | Address shown by the Tracking Streamer app on the AVP (step 4 of the checklist). Required for a connected AVP. |
| `--task drill_board` | Load the sim eval task, which spawns the task fixture and emits success/failure. `drill_board`, `grasp_drill` and `hammer_on_shelf` are the registered tasks; omit for a bare tabletop. Sim only. |
| `--record-dir data/drill_board` | Write episodes as `episode_NNNN.pt` under this directory, and arm the keyboard episode controls (see below). Without it nothing is saved. Sim only. |
| `--sim-cams` | Turn on the front sim camera so its images land in the recorded observations. Off by default — teleop itself does not need them, but data collection does. Add `--side-cam` for the second view; it is off by default because nothing downstream consumes it and it costs 2.8 MB per step. |
| `--camera-depth` | Add a depth channel to the front camera. Requires `--sim-cams`. |
| `--hand-tactile` | Record the full-hand Wuji tactile readings (`obs["hand_tactile"]`). Sim only, and the whole point of the visuotactile dataset. |
| `--tactile-layout glove` | Lay the touch sensors out like the Tachin glove: its 15 pads per hand plus the band across the top of the palm, each read back as the glove's own grid so sim and glove recordings line up. Default `band` keeps the older per-link band. Requires `--hand-tactile`. |

### Recorded episode layout

Each kept episode writes a single `episode_NNNN.pt` file. Camera RGB is stored as raw `uint8` frames directly inside it, alongside every other observation leaf (no separate video files; raw frames run ~2.8 MB/step and `torch.save` does not compress them). Depth is stored as `float16`. Neither changes the env's observation space — both are storage-only, applied as the episode is written.

Next to `obs` and `action`, the file has a `tracking` entry: the raw hand tracking from the device at every step, keyed by the operator's own hand (`left_hand` / `right_hand`, not swapped by `--mirror`). Each hand has `detected` (bool), `wrist_pose` (4×4, in the device's frame) and `keypoints` (21×3 joint positions relative to the wrist). It also has `head_pose` (4×4, same frame as the wrists; NaN when the device has no head tracking, e.g. WiLoR), so `inv(head_pose) @ wrist_pose` gives the wrist relative to the head. When a hand isn't seen, `detected` is false and the pose/keypoints are NaN. Step `t` of `tracking` is the reading that produced `action[t]`.

### Keyboard controls while recording

Keys are read from **the terminal that launched the script**, not the viewer window:

| Key | Effect |
|---|---|
| `k` | Keep the current episode (save `.pt`), then reset the scene. |
| `x` | Discard the current episode and reset the scene. |
| `q` | Quit. |

A task `terminated`/`truncated` (i.e. the task itself reported success or failure) counts as an implicit keep — the episode is saved and the scene resets automatically.

### Other flags worth knowing

Everything below has a working default; change them only if something feels wrong.

| Flag | Default | When to touch it |
|---|---|---|
| `--no-viewer` | viewer on | Run headless on a machine with no display. |
| `--front-cam-view` | robot-centred view | Switch the sim camera and the viewer back to the old calibrated third-person view. By default both sit on the robot's centre axis at 0.55 m looking at the table. Sim only. |
| `--no-table` | table on | Drop the table, clutter and task fixtures, leaving the robot alone in free space. For checking controller tracking and IK without anything in the way. Sim only; cannot be combined with `--task`. |
| `--table-rand` | same table every episode | By default every episode and every run uses one fixed table: calibrated size and height, dark grey legs, and the top set by `--table-texture` (default `wood_texture_1`; also `black_texture`, `stone_texture_1`–`4`, `wood_texture_1`–`8`, or an image path). Pass `--table-rand` to build a new random table at every reset instead (`--table-texture` is then ignored). Sim only. |
| `--show-eef-target` | off | Draw an axis triad at each arm's commanded palm pose in the Genesis 3D viewer, so the gap to the rendered hand is the tracking error. Sim only. The triads also land in rasterizer camera renders, so leave it off while recording camera observations. |
| `--mirror` | off | Bilateral mirror — your left hand drives the robot's right side. Natural when you stand facing the robot. |
| `--scale` | `1.0` | Scale operator hand translation into robot translation. Below 1 for finer control in a small workspace. |
| `--ema` / `--hand-ema` | `0.4` / `0.4` | Share of each new hand reading blended into the arm / finger targets. Lower for smoother, laggier motion; raise for snappier. |
| `--max-lin-vel` / `--max-rot-vel` | `0.25` m/s / `1.0` rad/s | Velocity clamps on the arm targets. |
| `--gesture-fist` / `--gesture-open` / `--gesture-window` | `1.25` / `1.55` / `2.5` s | Thresholds for the fist-then-open re-anchor gesture that engages/detaches an arm. `--no-reanchor-gesture` disables it. |
| `--no-arms` / `--no-hands` | both enabled | Freeze one half of the embodiment while debugging the other. |
| `--dt` | `1/20` s | Control period. With the viewer open the sim is held to real time, so this is also the collection rate. |
| `--steps` | `0` (unlimited) | Stop after N steps instead of running until interrupted. |
| `--log-hz` | `1.0` | Rate of the per-side status line (`active`, `engaged`, grip ratio, xyz, mean finger flex). |
| `--offline` | off | Build the env without connecting to any device — smoke-testing the pipeline. |

`--real`-only flags (`--left-hand-serial`, `--right-hand-serial`, `--hand-filter-hz`, `--hand-max-temp`, `--hand-warn-temp`) and the WiLoR capture/camera-axis flags (`--camera-id`, `--width`, `--height`, `--focal-length`, `--camera-x`, `--camera-y`, `--alignment`, `--zed-serial`) do not apply to AVP sim collection; the script rejects them in that combination. Run `python scripts/teleop.py --help` for the full list.

### Scripted collection (no operator)

`scripts/scripted_grasp_drill.py` in the sibling `vtdex_policies` repo (`../vtdex_policies`, installed by the requirements file; the policy itself is `vtdex_policies.policy.scripted.GraspDrillPolicy`) collects the `grasp_drill` task without a device: a scripted right hand grips the drill handle with the index finger on the trigger, squeezes, and lifts it 15 cm. From inside `../vtdex_policies/`:

```bash
python scripts/scripted_grasp_drill.py --episodes 50 --record-dir data/grasp_drill \
    --sim-cams --camera-depth --hand-tactile
```

Each successful episode is saved as `episode_NNNN.pt` in the layout above, without the `tracking` entry (there is no operator), plus `scripted_policy` and `success` fields. Failed episodes are dropped unless `--keep-failures` is given. `--no-viewer`, `--front-cam-view`, `--tactile-layout`, `--table-texture` and `--dt` (default `1/20` s) work as in `teleop.py`; `--seed` seeds the first reset. The drill spawns at the same fixed pose every episode and the policy has no randomness, so the robot and drill motion come out near-identical from episode to episode; only the background lighting changes.

## Working in a submodule

Submodules start in detached HEAD at the pinned commit. To do work, create your own branch both in the meta-repo and in the submodule you are using. 

```bash

git checkout -b <your own meta-repo branch name>
cd genesis_adaptor
git checkout -b <your own repo branch name>     

```
