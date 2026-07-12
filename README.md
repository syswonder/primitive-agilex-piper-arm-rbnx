# primitive-agilex-piper-arm-rbnx

Robonix package wrapping the **AgileX Piper 6-DoF arm** hardware driver. Owns the `primitive/arm/*` namespace.

Catalog name: `robonix.primitive.agilex.piper.arm`.

> Naming note: internal directory / binary name is `piper_ctl` (`_ctl` = low-level control), while the contract namespace is `arm`. Piper is an arm, so calling this `piper_chassis_rbnx` would mislead.

## Capability surface

| Contract                                | Mode       | Transport | Source / handler                                     |
| --------------------------------------- | ---------- | --------- | ---------------------------------------------------- |
| `robonix/primitive/arm/driver`          | rpc        | gRPC      | `Driver(CMD_INIT, config_json)` — lifecycle gate     |
| `robonix/primitive/arm/joint_states`    | topic_out  | ROS 2     | `/<ns>/joint_states_single` (sensor_msgs/JointState) |
| `robonix/primitive/arm/arm_status`      | topic_out  | ROS 2     | `/<ns>/arm_status` (piper_msgs/PiperStatusMsg)       |
| `robonix/primitive/arm/end_pose`        | topic_out  | ROS 2     | `/<ns>/end_pose` (geometry_msgs/Pose)                |
| `robonix/primitive/arm/pos_cmd`         | topic_in   | ROS 2     | `/<ns>/pos_cmd` (piper_msgs/PosCmd) — driver subscribes; consumers publish |

All five contracts are **package-locally defined** (see `capabilities/primitive/arm/*.v1.toml`) because the robonix global tree does not ship `primitive/arm/*` yet (it has chassis / camera / lidar / imu / audio). The two vendor-specific message types (`PiperStatusMsg`, `PosCmd`) are also shipped at package level, at `capabilities/lib/piper_msgs/msg/*.msg`.

> **Single-source-of-truth** for the two `.msg` IDLs is the vendored `src/piper_msgs/msg/` directory. The mirrored copies under `capabilities/lib/piper_msgs/msg/` are what `rbnx codegen` / atlas's contract registry actually scan. Keep them in sync — if upstream `piper_msgs` ever changes, update both. Plain file copies are used rather than symlinks to keep `git diff` legible.

## Boot ordering

Boot this **before** any consumer of `primitive/arm/*`. In the vertical-grasp pipeline:

- `primitive-agilex-piper-description-rbnx` consumes `arm/joint_states` to drive `robot_state_publisher`;
- `service-piper-moveit-rbnx` consumes `arm/arm_status` and publishes to `arm/pos_cmd`;
- `skill-pick-rbnx` polls `arm/arm_status` between grasps.

rbnx-cli has no defer/retry, so providers MUST come first in YAML declaration order.

## Driver-init lifecycle

`start.sh` brings up the atlas bridge — no ROS spawn. The bridge opens a gRPC server, registers the provider, declares only `primitive/arm/driver` (auto-emitted by the framework when codegen produces a `Driver` Servicer), then blocks on `Driver(CMD_INIT, config_json)`.

When `rbnx boot` invokes Init it passes the manifest's `config:` block as JSON. The handler:

1. validates cfg (CAN port, bitrate, gripper flags, sentinel timeout);
2. optionally runs `scripts/can_activate.sh` (when `auto_can_setup=true`);
3. spawns `ros2 launch piper start_single_piper.launch.py …`;
4. waits for the first `sensor_msgs/JointState` on `/<ns>/joint_states_single` as proof the CAN link came up;
5. declares `arm/joint_states`, `arm/arm_status`, `arm/end_pose`, `arm/pos_cmd` on atlas, and returns ok.

`CMD_DEACTIVATE` / `CMD_SHUTDOWN` kill the piper subprocess. Idempotent.

## CAN bring-up — TWO paths

The Piper arm uses USB-CAN (MCP251xFD or similar). The CAN interface must be **up** and named `can_piper` (or whatever you set `can_port:` to) before `piper_ctrl_single_node` can talk to it. The CAN interface name is config-driven — the package never hardcodes a device name.

### Path A — recommended for production (default)

Bring `can_piper` up **outside `rbnx boot`**, ahead of time. On a Jetson with udev rules, this is once-per-boot:

```bash
# Run once per host boot, BEFORE `rbnx boot`:
bash scripts/can_activate.sh can_piper 1000000 "1-4.2:1.0"
# Replace "1-4.2:1.0" with the USB bus path of your Piper.
# To list candidates:
#   lsusb -t
#   sudo ethtool -i can0 | grep bus-info
```

`auto_can_setup` stays `false` in the manifest. This path keeps the deploy free of sudo coupling — the operator approves the sudo prompt once, interactively, then `rbnx boot` runs unattended.

### Path B — convenience for dev laptops

Set `auto_can_setup: true` in the manifest config block. `on_activate` will run `scripts/can_activate.sh` itself before spawning the driver. Requires **passwordless sudo** for the operator (otherwise the script blocks on the prompt and `Driver(CMD_INIT)` times out).

```yaml
- name: piper_ctl
  path: ../packages/piper_ctl_rbnx
  config:
    can_port:        can_piper
    can_bitrate:     1000000
    can_usb_address: "1-4.2:1.0"
    auto_can_setup:  true
```

Path B is more convenient when the USB-CAN's bus path shifts between sessions, but couples the deploy to sudoers state — not ideal for production.

## Layout

```
primitive-agilex-piper-arm-rbnx/
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

To drive Init manually (without `rbnx boot`): from any robonix gRPC client, call the arm's `Driver` service with `command=0` (CMD_INIT) and a JSON config blob. The handler returns `ok=true` after the first JointState is observed, then declares the four data topics.

## Verification

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

`src/piper/`, `src/piper_msgs/`, `src/graspnet_msgs/` are verbatim copies from [agilexrobotics/piper_ros](https://github.com/agilexrobotics/piper_ros). The other packages in the upstream workspace are deliberately **not** vendored here and land in their own robonix packages:

- `piper_description` / `piper_with_gripper_moveit` → `primitive-agilex-piper-description-rbnx`.
- `piper_humble` / `piper_moveit_control` → `service-piper-moveit-rbnx`.
- `piper_gazebo` / `piper_mujoco` / `piper_sim` → not migrated (sim-only).

If anything diverges from upstream, drop a `*.patch` alongside `src/` documenting the diff.

## License

This package: Apache-2.0. Vendored piper_ros / piper_msgs / graspnet_msgs: see their respective LICENSE files.
