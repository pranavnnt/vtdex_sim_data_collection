# Collecting `grasp_drill` data

Teleoperated demonstrations of the right Wuji hand picking up a power drill by its handle and lifting it.

## 1. Before you start

1. Install everything as described in [Install instructions](../README.md#install-instructions) in the main README.
2. Work through the [Pre-data collection checklist](../README.md#pre-data-collection-checklist): WiFi on the sim PC, AVP charged and calibrated, both on RedRover, Tracking Streamer open, AVP IP noted.
3. From inside `tianji_sim_real/`, start collection:

```bash
python scripts/teleop.py --sim --device avp --ip <AVP IP> --task grasp_drill \
    --record-dir data/grasp_drill --sim-cams --camera-depth --hand-tactile
```

The flags are explained in [What each flag in that command does](../README.md#what-each-flag-in-that-command-does). Episodes are written to `data/grasp_drill/episode_NNNN.pt`; numbering continues from whatever is already in the folder.

## 2. The task

### Start of every episode

Every episode starts from exactly the same scene:

- **Table:** one fixed table (wood top, dark grey legs), no other objects on it.
- **Drill:** the YCB power drill stands upright on its battery pack, 63 cm in front of the robot and 21 cm to its right. The bit points forward-left, so the handle's back faces the robot's right hand.
- **Right arm:** starts low over the table, just behind and to the right of the drill handle, palm upright (thumb up) and turned towards the handle, fingers open.
- **Left arm and hand:** at their home pose. They are not used in this task.
- **Viewer:** zoomed onto the drill and right hand. The recorded camera keeps the full view.

### Success

The episode succeeds as soon as the drill is **10 cm above where it started**. The episode is then saved automatically and the scene resets.

There is no time limit and no automatic failure. If the drill falls over or the attempt goes wrong, you have to discard the episode yourself (`x`, see below).

## 3. What you do

1. **Wait for the viewer**, then look at the terminal: it prints `recording armed: [k]eep episode  [x] discard  [q]uit`.
2. **Take control of the right arm** with your right hand: make a fist, open, fist, open (within 2.5 s). The terminal prints `engaged right (gesture)`. The arm does not jump: from now on it copies how your hand *moves*, starting from where it already is. Leave your left hand alone so the left arm stays still.
3. **Grasp the handle:** bring the palm onto the back of the handle, wrap your fingers round its right side to the front, index finger on the trigger, then close your hand firmly.
4. **Lift straight up**, steadily, until the drill is 10 cm up. The episode saves and resets by itself.
5. **Next episode:** the scene resets to the same start and the arm is released. Take control again with fist-open-fist-open (step 2).

**Do not let go of the arm mid-episode.** The same fist-open-fist-open gesture, or losing hand tracking, releases the arm: it drives back to the robot's home pose and the hand opens, dropping the drill. If that happens, discard the episode.

### Keys

Keys are read from **the terminal that launched the script**, not the viewer, so keep that terminal focused.

| Key | Effect |
|---|---|
| `k` | Keep the current episode, then reset. Only needed if you want to save an episode that did not reach 10 cm. |
| `x` | Discard the current episode and reset. Use this whenever the drill falls or the grasp goes wrong. |
| `q` | Quit. |

## 4. Teleop settings

These change how the robot follows your hand. The defaults are the current best guess; change them on the command line if the arm feels sluggish, shaky or too fast.

| Flag | Default | What it does | Change it when |
|---|---|---|---|
| `--scale` | `1.0` | How far the robot palm moves for each cm your hand moves. | Lower (e.g. `0.7`) for finer positioning around the handle; raise if you run out of reach. |
| `--ema` | `0.4` | How much of each new hand reading the arm takes (0–1). Lower is smoother but lags more. | Raise if the arm feels slow to follow; lower if it shakes. |
| `--hand-ema` | `0.4` | Same as `--ema`, for the fingers. | Raise if the fingers close too slowly; lower if they twitch on the handle. |
| `--max-lin-vel` | `0.25` m/s | Top speed of the palm. | Raise if the arm falls behind quick moves; lower if it knocks the drill over on approach. |
| `--max-rot-vel` | `1.0` rad/s | Top turning speed of the wrist. | Raise if the wrist lags when you turn your hand. |
| `--dt` | `1/20` s | Time per control step, which is also the recording rate (20 Hz by default). | Lower for a more responsive arm, only if the sim keeps up. Keep it the same across a dataset. |
| `--activation-frames` | `3` | Steps of steady hand tracking needed before the arm engages. | Raise if the arm engages by accident. |
