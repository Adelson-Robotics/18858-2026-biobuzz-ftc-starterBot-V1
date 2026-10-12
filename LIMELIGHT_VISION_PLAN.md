# Limelight Vision Plan: AprilTag Pose Correction

**Status:** plan only, nothing built yet. Work through the steps in order: each one is checked before the next one
depends on it.

## Why we're doing this

The robot knows where it is from odometry (the Pinpoint and its two pods). Odometry drifts, and it can skid badly
when the robot hits a wall. In one 10-09 run the robot thought it was 30 inches off the field. Auto-aim trusts that
position, so a bad position means a missed shot. The Limelight 3A (already mounted, tilted about 30° up) can see the
HIVE AprilTags and tell us where the robot really is.

### Rules for this game (BIOBUZZ)

- There are **16 AprilTags: 4 on each HIVE cell**. The tags are fixed to the cells and move with them.
- **Only use tags on a cell that is tipped UP.** When a cell is up, its tags face about 30° downward, almost
  square-on to our upward-tilted Limelight. When a cell is down, its tags can't be seen from shooting spots, only
  from underneath the hive, and their position is different, so they would give a wrong answer.
- **Correct the robot's position every time an up cell is seen**, as long as the reading passes the bad-data filter
  (step 6).
- **Flower_Pause autos only:** shoot our 4 balls only if our target cell is up (the other team has made their shots).
  If it isn't up, keep the balls and just park. NoPause autos and TeleOp shooting stay as they are now.

## What last year's code did (18858-EveryBot-Code)

See `Limelight/LimelightPoseUpdater.java` and `Teleop/BLUE.java` in
[Adelson-Robotics/18858-EveryBot-Code](https://github.com/Adelson-Robotics/18858-EveryBot-Code).

- The Limelight did all the math itself ("MegaTag"). The camera's position on the robot and the tag map were typed
  into the **Limelight's web setup page**, not into our code. The code just read `result.getBotpose()`.
- That position is in meters, measured from the field center. It was converted to Pedro coordinates with
  `pedroX = y × 39.37 + 72` and `pedroY = −x × 39.37 + 72`. Heading was kept from the Pinpoint.
- The position was **replaced outright** (`follower.setPose()`) only when the reading was valid, at least one tag
  was seen, the robot was nearly still (under 1.8 in/s and 1.8 rad/s), and it wasn't following a path. Changing the
  position mid-path makes the path follower jerk.
- There's a Kalman filter class in that file, but **it was never actually used**.

## Our approach (what's different this year)

**We'll do the math in our own code instead of on the Limelight's web page.** The Limelight will just report each tag
it sees: its ID, and where that tag is relative to the camera. Our code holds the camera's position on the robot and
the tag map as constants. That's better for us because:

- it's saved in git with the rest of the code, so everyone can see it and it can't silently change on the camera;
- it can be unit-tested on a laptop, like the shot math already is;
- we can check **each tag's angle** to decide whether its cell is up. Last year's combined all-tags answer couldn't
  do that one tag at a time.

Heading will still come from the Pinpoint, which is reliable. The Limelight only corrects x and y.

## The steps

| # | Step | What it produces | How we check it |
|---|------|------------------|-----------------|
| 1 | **Set up the Limelight** (no robot code). Add a `Limelight3A` named `limelight` to the Control Hub's robot configuration. On the Limelight's web page, make an AprilTag pipeline with the game's tag family and tag size (from the CAD) and turn on full 3D. | A Limelight that detects the HIVE tags | The web page shows the tags and their IDs |
| 2 | **Measure where the camera is on the robot.** Measure from the **robot's center**: that's the point the Pedro position tracks (the Pinpoint pod offsets in `subsystems/drivetrain/pedro/Constants.java` are measured from it). Measure forward, left and up in inches, plus the tilt angles: pitch (about 30° up), yaw and roll. | Camera-offset constants | Checked in step 4 |
| 3 | **Build the tag map from the field CAD.** For all 16 tags, with each cell **up**: tag ID, which cell it's on, where its center is, and which way it faces. Convert these to Pedro coordinates once and store them in one table in code. | The tag map | The tags should sit around our four known cell positions in `ShotTarget.java`. Also confirm the field size: our code assumes 141.5 in (center 70.75), last year's assumed 144 in (center 72). |
| 4 | **Read-only test OpMode.** Show and log every tag it sees (ID, distance, angle), the robot position each tag gives, and the odometry position side by side. **It doesn't change the robot's position yet.** | A way to see whether the math is right | Park the robot at known spots (the auto start positions against the walls). The tag position should match within about 1–2 inches. |
| 5 | **"Is this cell up?" check.** Accept a tag only if it looks nearly square-on to the camera (how an up tag looks) and is a sensible distance away. This check doesn't use the odometry, which is the thing we're trying to fix. | The up check | Step-4 logs: up cells pass. Driving under a down cell should fail the check. |
| 6 | **Bad-data filter, then blend.** Accept a reading only if: the up check passes; the robot is moving slowly (start from last year's 1.8 in/s and 1.8 rad/s); the reading is fresh, not delayed; and no path is running. If a reading disagrees with the odometry by a lot, only believe it after several frames in a row agree. That way we still recover from a real 30-inch skid but ignore one bad frame. Then blend it in with last year's Kalman filter (actually used this time), for x and y only. | Trusted position corrections | Unit tests, then logs |
| 7 | **Put it in the code structure** (see `CLAUDE.md`). A new `subsystems/vision/` subsystem owns the Limelight, the camera offset, the tag map and the up check. It reports accepted corrections and which cells it has seen up. `Robot.update()` passes the corrections to `Drivetrain`, which still keeps the Pedro follower private. The math goes in pure classes (like `IndexerMath`) with unit tests. | Clean, testable code | Builds; unit tests pass |
| 8 | **Turn the correction on** in TeleOp and the autos. Add vision columns to the CSV logs. Optional: clear the "ODOMETRY OFF" warning after a good correction. | Position fixes during matches | Logs show corrections only when an up cell is in view, with no sudden jumps |
| 9 | **Flower_Pause rule.** During its 15-second wait (the robot is sitting still, which is ideal for this), watch our **target cell**: red shoots RED_UPPER, blue shoots BLUE_LOWER from the start spot. After the wait: if that cell was seen up, aim and shoot as now; if not, skip shooting and park. Only `MatchAuto1`; NoPause doesn't change. | The Flower_Pause rule | Run it with the cell up (it shoots) and with the cell down (it parks with all 4 balls) |

## Files involved (when we build it)

- **New:** `subsystems/vision/` (Limelight subsystem, tag map, pose math and up check), a test OpMode, and unit tests
  in `tests/vision/`.
- **Changed:** `Robot.java` (owns the vision subsystem and passes corrections along),
  `subsystems/drivetrain/Drivetrain.java` (a method to apply a correction), `opmodes/autonomous/match/MatchAuto1.java`
  (the Flower_Pause rule; maybe `MatchAutoBase.java` too), `opmodes/teleop/MatchTeleOp.java` (telemetry and logging).
- **Reused:** the cell positions in `ShotTarget` (step 3 check), `CsvLog` (logging), and last year's Kalman filter
  logic.

## What we need from the team before step 3

- [ ] The AprilTag **family** and **tag size**
- [ ] All **16 tag positions and facing directions** from the field CAD, with the cells **up**
- [ ] Which **cell each tag ID** belongs to
- [ ] The camera offset from step 2 (forward, left, up, and the tilt angles)
