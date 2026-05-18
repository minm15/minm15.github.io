# ROS2 Bag

`ros2 bag` is the standard ROS 2 tool for recording, inspecting, and replaying topic traffic.

## Common Commands

Record selected topics:

```bash
ros2 bag record /camera/image_raw /imu/data
```

Record everything:

```bash
ros2 bag record -a
```

Inspect a bag:

```bash
ros2 bag info <bag-directory>
```

Replay a bag:

```bash
ros2 bag play <bag-directory>
```

## Operational Notes

- Recording all topics is convenient but can produce large datasets quickly.
- Clock behavior matters when replaying into systems that depend on simulated time.
- Storage plugins affect performance and file layout.
