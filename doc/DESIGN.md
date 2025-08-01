# Design

The package exposes a single node `NavigatorNode` that converts camera images into velocity commands. It is intended for the ET Robocon robot and assumes a camera mounted at the front of the vehicle.

## Node overview
- **Subscription**: `/camera_top/camera_top/image_raw` (`sensor_msgs/Image`)
- **Publication**: `/cmd_vel` (`geometry_msgs/Twist`)

## Algorithm
1. Convert the input image to grayscale and apply a binary threshold so the dark line appears white.
2. For each predefined scan line, detect connected white pixel blobs.
   Select the blob whose center is closest to the previously detected line
   position (or the image center if none is available).
    While a scan line is in `blue_to_black`, each frame checks all detected
    blobs to determine if a right-hand branch exists. A branch decision
    requires at least two blobs. Blobs narrower than `MIN_BLOB_WIDTH` (5 px)
    are ignored. Those within `BRANCH_WINDOW` (40 px) of the previous center
    are considered and the right-most among them is chosen. If no candidate
    falls inside the window, the closest blob to the previous center is used.
    When a valid branch is chosen, the scan line immediately returns to
    `normal` and the selected center overrides all scan lines for that frame.
    During the branch-following phase (after a confirmed `blue_to_black`),
    for the next few scan lines (while `pending_branch` &gt; 0), each selects
    the rightmost blob candidate regardless of distance to the previous center,
    ensuring persistent right-hand branch tracking. Once `pending_branch`
    expires, normal proximity-based selection resumes.
    While a scan line is in `blue_detected` or
    `blue_to_black`, its
    chosen center immediately replaces the reference center for the next lines
    so the blob ranking relies on the latest estimate. Each scan line tracks a
    small state machine (`normal`, `blue_detected`, `blue_to_black`) to report
    if a blue area temporarily occludes the line.
3. Calculate the weighted deviation of these center positions from the image center.
4. Convert this deviation into an angular velocity using a proportional gain.
   The linear velocity is scaled using the current angular velocity and
   smoothed by a low-pass filter.
5. Publish the resulting `Twist`.

## Parameters
- `scan_lines`: list of normalized y positions (ratio of image height) used for line detection.
- `weights`: weight value for each scan line.
- `operation_gain`: gain to transform the deviation into an angular velocity.
- `min_linear`: minimum linear velocity.
- `max_linear`: maximum linear velocity.
- `max_angular`: maximum angular velocity.
- `alpha`: coefficient for the low-pass filter used on linear velocity.
- `debug`: enable OpenCV visualization when set to `true`.

The node retrieves the image width and height from each received frame, so it can adapt to different camera resolutions.
