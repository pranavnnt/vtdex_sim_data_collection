# vtdex_sim_data_collection

Meta-repo pinning the set of repositories used for VTDex simulation data collection.
It contains no code of its own — each directory is a git submodule tracking an
independent upstream repository at a specific commit.

| Path | Upstream | Branch | Role |
|---|---|---|---|
| `genesis_adaptor/` | `UniEnvOrg/genesis_adaptor` | `tactile_sensing` | Genesis physics engine → UniEnv adaptor: scenes, robots, sensors, controllers, reference policies |
| `tianji_sim_real/` | `realquantumcookie/tianji_sim_real` | `tactile_sensing` | Sim and real env composers for the Tianji Marvin dual-arm robot + Wuji dexterous hands |
| `unienv/UniEnv/` | `UniEnvOrg/UniEnv` | `main` | Core UniEnv framework (`World`, `WorldNode`, `Env`, spaces, compute backends) |
| `unienv/unienv-input-devices/` | `UniEnvOrg/unienv-input-devices` | `main` | Teleop input devices (AVP, WiLoR, SpaceMouse) |
| `unienv/unienv-tianji/` | `UniEnvOrg/unienv-tianji` | `main` | Tianji arm hardware driver |
| `unienv/unienv-wuji/` | `UniEnvOrg/unienv-wuji` | `main` | Wuji dexterous hand hardware driver |

## Clone

```bash
git clone --recurse-submodules <this-repo-url>
```

Already cloned without submodules:

```bash
git submodule update --init --recursive
```

## Working in a submodule

Submodules start in detached HEAD at the pinned commit. To do work:

```bash
cd genesis_adaptor
git checkout tactile_sensing     # branch named in .gitmodules
# ... commit and push as normal, inside the submodule ...
```

Then record the new pin in this meta-repo:

```bash
cd ..
git add genesis_adaptor
git commit -m "Bump genesis_adaptor"
```

Push the submodule's own commits **before** pushing the updated pin, or the
pin will not resolve for anyone else.

## Updating all submodules to their latest upstream

```bash
git submodule update --remote --merge
git add -A && git commit -m "Bump submodules"
```
