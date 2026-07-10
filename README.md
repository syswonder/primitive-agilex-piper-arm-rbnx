# piper_ctl_rbnx

Robonix package wrapping the **AgileX Piper 6-DoF arm** hardware
driver. Owns the `primitive/arm/*` namespace for the `piper_grasp`
deploy.

> Naming note: this package is `piper_ctl_rbnx` (not `piper_chassis_rbnx`)
> — Piper is an arm, "chassis" would mislead. `_ctl` = low-level
> control, the contract namespace is `arm`. See MIGRATION_PLAN.md
> §2.0 for context.

## Boot ordering

This is the **arm primitive** for the piper_grasp deploy. Boot it
**before** any consumer of `primitive/arm/*` — Stage 3A
`piper_description_rbnx` consumes `arm/joint_states` to drive
`robot_state_publisher`; Stage 5 `piper_moveit_rbnx` consumes
`arm/arm_status` and publishes to `arm/pos_cmd`; Stage 6
`pick_skill_rbnx` polls `arm/arm_status` between grasps. rbnx-cli
has no defer/retry so providers MUST come first in YAML
declaration order.

## Capability surface

| Contract                                | Mode      | Transport | Source / handler                                    |
| --------------------------------------- | --------- | --------- | --------------------------------------------------- |
| `robonix/primitive/arm/driver`          | rpc       | gRPC      | `Driver(CMD_INIT, config_json)` — lifecycle gate    |
| `robonix/primitive/arm/joint_states`    | topic_out | ROS 2     | `/<ns>/joint_states_single` (sensor_msgs/JointState) |
| `robonix/primitive/arm/arm_status`      | topic_out | ROS 2     | `/<ns>/arm_status` (piper_msgs/PiperStatusMsg)      |
| `robonix/primitive/arm/end_pose`        | topic_out | ROS 2     | `/<ns>/end_pose` (geometry_msgs/Pose)               |
| `robonix/primitive/arm/pos_cmd`         | topic_in  | ROS 2     | `/<ns>/pos_cmd` (piper_msgs/PosCmd) — driver subscribes; consumers publish |

All five contracts are **package-locally defined** (see
`capabilities/primitive/arm/*.v1.toml`) because the robonix global
tree doesn't ship `primitive/arm/*` yet (it has chassis / camera /
lidar / imu / audio). codegen + atlas merge package-level contracts
automatically. The two vendor-specific message types
(`PiperStatusMsg`, `PosCmd`) are also shipped at package level, at
`capabilities/lib/piper_msgs/msg/*.msg`.

> **single-source-of-truth** for the two `.msg` IDLs is the vendored
> `src/piper_msgs/msg/` directory. The mirrored copies under
> `capabilities/lib/piper_msgs/msg/` are what `rbnx codegen` /
> atlas's contract registry actually scan. Keep them in sync — if
> upstream piper_msgs ever changes, update both. (We use plain
> file copies rather than symlinks to keep `git diff` legible.)

## Driver-init lifecycle

`start.sh` brings up the atlas bridge (Python). The bridge registers
the provider, declares only `primitive/arm/driver` (auto-emitted by
the framework when codegen produces a `Driver` Servicer), then blocks
on `Driver(CMD_INIT, config_json)`.

When `rbnx boot` invokes Init it passes the manifest's `config:`
block as JSON. The handler validates cfg, optionally runs
`scripts/can_activate.sh` (when `auto_can_setup=true`), spawns
`ros2 launch piper start_single_piper.launch.py …`, waits for the
first `JointState` on `joint_states_single` as proof the CAN link
came up, declares the four ROS 2 topics on atlas, and returns ok.

## CAN bring-up — TWO paths

The Piper arm uses USB-CAN (MCP251xFD or similar). The CAN interface
must be `up` and named `can_piper` (or whatever you set
`can_port:` to) before `piper_ctrl_single_node` can talk to it.
There are two ways to make that happen:

If the configured USB address or interface name looks wrong, inspect
the current host mapping first:

```bash
bash scripts/find_all_can_port.sh
```

Use the reported USB bus path, for example `1-4.2:1.0`, as
`can_usb_address`. Do not guess when multiple CAN interfaces are
present; Jetson onboard `*.mttcan` interfaces and USB-CAN adapters can
coexist.

### Path A — recommended for production (default)

Bring `can_piper` up **outside `rbnx boot`**, ahead of time. On a
Jetson with udev rules, this is once-per-boot:

```bash
# Run once per host boot, BEFORE `rbnx boot`:
cd /Users/howenliu/lab/packages/piper_ctl_rbnx
bash scripts/can_activate.sh can_piper 1000000 "1-4.2:1.0"
# Replace "1-4.2:1.0" with the USB bus path of your Piper.
# To list candidates:
#   lsusb -t
#   sudo ethtool -i can0 | grep bus-info
```

`auto_can_setup` stays `false` in the manifest. This path keeps the
deploy free of sudo coupling — the operator approves the sudo prompt
once, interactively, then `rbnx boot` runs unattended.

### Path B — convenience for dev laptops

Set `auto_can_setup: true` in the manifest config block. `on_activate`
will run `scripts/can_activate.sh` itself before spawning the
driver. Requires **passwordless sudo** for the operator (otherwise
the script blocks on the prompt and `Driver(CMD_INIT)` times out).

```yaml
- name: piper_ctl
  path: ../packages/piper_ctl_rbnx
  config:
    can_port: can_piper
    can_bitrate: 1000000
    can_usb_address: "1-4.2:1.0"
    auto_can_setup: true
    # ...
```

Path B is more convenient when the USB-CAN's bus path shifts between
sessions, but couples the deploy to sudoers state — not ideal for
production.

## Layout

```
piper_ctl_rbnx/
├── package_manifest.yaml
├── capabilities/
│   ├── primitive/arm/
│   │   ├── driver.v1.toml
│   │   ├── joint_states.v1.toml
│   │   ├── arm_status.v1.toml
│   │   ├── end_pose.v1.toml
│   │   └── pos_cmd.v1.toml
│   └── lib/piper_msgs/msg/                  # IDL for codegen
│       ├── PiperStatusMsg.msg               # mirror of src/piper_msgs/msg/
│       └── PosCmd.msg
├── piper_ctl/
│   ├── __init__.py
│   └── main.py                              # lifecycle + sentinel
├── scripts/
│   ├── build.sh                             # colcon + rbnx codegen
│   ├── start.sh                             # source ROS, exec main
│   ├── find_all_can_port.sh                 # print CAN interface ↔ USB bus mapping
│   └── can_activate.sh                      # vendored from upstream piper_ros
└── src/                                     # vendored (no .git, no build/install)
    ├── piper/                               # main ROS 2 driver (rclpy)
    ├── piper_msgs/                          # PiperStatusMsg + PosCmd + Enable.srv
    └── graspnet_msgs/                       # GraspPose.msg — required by piper/package.xml
```

## Config (passed via `Driver(CMD_INIT, config_json)`)

```yaml
can_port:            can_piper        # CAN interface name
can_bitrate:         1000000
can_usb_address:     ""               # pin via USB bus path; "" = first detected
auto_can_setup:      false            # see "CAN bring-up" above
auto_enable:         true             # forwarded to start_single_piper auto_enable
gripper_exist:       true
gripper_val_mutiple: 2                # upstream typo preserved
arm_namespace:       /arm             # see note in main.py docstring
sentinel_timeout_s:  30.0             # max wait for first JointState in on_activate
# Topic-name overrides (rarely needed; default derives from arm_namespace):
# joint_states_topic: /arm/joint_states_single
# arm_status_topic:   /arm/arm_status
# end_pose_topic:     /arm/end_pose
# pos_cmd_topic:      /arm/pos_cmd
```

## Build / run standalone

```bash
bash scripts/build.sh                                  # colcon + rbnx codegen
ROBONIX_ATLAS=127.0.0.1:50051 \
    bash scripts/start.sh                              # registers, awaits Init
```

To drive Init manually (without `rbnx boot`): from any robonix gRPC
client, call the arm's `Driver` service with `command=0` (CMD_INIT)
and a JSON config blob. The handler returns `ok=true` after the
first JointState is observed, then declares the four data topics.

## Verification (Stage 2 deliverable)

After `rbnx boot` from `piper_grasp_deploy/`:

```bash
rbnx caps | grep arm
# Expected: piper_ctl provider with
#   robonix/primitive/arm/{driver, joint_states, arm_status, end_pose, pos_cmd}

ros2 topic hz /arm/joint_states_single             # ~200 Hz
ros2 topic echo /arm/arm_status --once             # PiperStatusMsg fields
ros2 topic echo /arm/end_pose --once               # geometry_msgs/Pose

# Reverse cmd path (CAREFUL — moves the arm; clear the workspace first):
ros2 topic pub --once /arm/pos_cmd piper_msgs/msg/PosCmd \
    "{x: 0.30, y: 0.0, z: 0.25, roll: 0.0, pitch: 1.57, yaw: 0.0,
      gripper: 0.05, mode1: 0, mode2: 0}"
```

## Vendor / upstream

`src/piper/`, `src/piper_msgs/`, `src/graspnet_msgs/` are verbatim
copies from
[agilexrobotics/piper_ros](https://github.com/agilexrobotics/piper_ros)
at the version that worked on the original Jetson with
`/Users/howenliu/lab/grasp/driver/piper_ros/`. The other packages
in the upstream workspace (`piper_humble`, `piper_with_gripper_moveit`,
`piper_moveit_control`, `piper_description`, `piper_gazebo`,
`piper_mujoco`, `piper_no_gripper_moveit`, `piper_sim`) are
deliberately **not** vendored here:

- `piper_description` / `piper_with_gripper_moveit` → Stage 3A
  `piper_description_rbnx` (URDF + robot_state_publisher).
- `piper_humble` / `piper_moveit_control` → Stage 5
  `piper_moveit_rbnx`.
- `piper_gazebo` / `piper_mujoco` / `piper_sim` → not migrated
  (sim-only; original deploy doesn't use them).

If anything diverges from upstream, drop a `*.patch` alongside
`src/` documenting the diff.

## License

This package: Apache-2.0 (matches piper_ros upstream).
