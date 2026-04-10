# AAE5303 Robust Control Technology in Low-Altitude Aerial Vehicle

## Post-Lesson Reflection Report

|  |  |
|---|---|
| **Student Name:** | HOU Guanhao |
| **Student ID:** | 25039773G |
| **Group Number:** | 08 (GOGOGO) |
| **Date:** | 11/04/2026 |

---

## Section 1: AI Usage Experience

I used Cursor AI assistant for nearly every stage of the ORB-SLAM3 project. I used it very frequently — almost every coding and debugging session throughout the whole project.

**Key tasks:** The AI configured the Docker container with volume mounts (`-v D:\:/data`), wrote `ros_mono_compressed.cc` with CLAHE preprocessing and trajectory saving, patched `System.cc` to bypass monocular restrictions, designed four YAML configs (v1–v4) with different ORB parameters, generated evaluation scripts (`evaluate.sh`, `extract_ground_truth.py`), and created visualization figures with matplotlib.

**Most useful feature:** Terminal command execution was the most helpful — the AI could directly run Docker commands, compile C++ code inside the container, and execute `evo` evaluation tools. The chat feature was also useful for explaining ORB-SLAM3 source code structure and suggesting what parameters to tune next.

---

## Section 2: Understanding AI Limitations

**Example 1 — Wrong shell syntax:** The AI frequently used Linux syntax like `2>/dev/null` in Windows PowerShell, causing parser errors every time. It assumed a Linux environment but my host system runs Windows. The fix was to wrap commands inside `bash -c "..."` when running in Docker. This happened many times and the AI kept making the same mistake.

**Example 2 — Wrong IMU extrinsic:** When setting up Mono-Inertial mode, the AI copied a LiDAR-to-Camera transformation matrix as the IMU-to-Camera extrinsic (`T_b_c1`). These are completely different sensors mounted at different positions on the drone. The Mono-Inertial mode failed repeatedly with "bad imu flag" errors. I detected this by carefully reading the dataset calibration documentation and realizing the matrix was labeled for LiDAR, not IMU.

**Example 3 — Completeness over 100%:** The AI's pose coverage script reported 100.69% completeness — clearly impossible since completeness cannot exceed 100%. The bug was that multiple estimated poses could match the same ground truth pose, inflating the count. I spotted the error immediately and asked for a fix using a set of unique GT indices. The corrected result was 94.51%.

---

## Section 3: Engineering Validation

**Multiple configurations:** We tested four versions (v1–v4) with different feature counts, FAST thresholds, and playback rates to confirm improvements were consistent, not random.

**Ground truth comparison:** All evaluations used `evo` with Sim(3) alignment and scale correction against RTK GPS ground truth. We checked matched pose counts, scale factors, and error distributions.

**Sanity checks:** When v2 reported scale factor 2.204 (trajectory at half real size) and v3 reported 1.301 (closer to real), both were consistent with expected monocular SLAM behavior. When `CameraTrajectory.txt` was missing after the first run, this led us to discover the System.cc restriction.

---

## Section 4: Problem-Solving Process

**The issue:** Our plan included Mono-Inertial mode as the most advanced tier. The AMtown02 dataset has 400Hz IMU data, which should improve scale estimation. We prepared `AMtown02_MonoInertial.yaml` and a new ROS node.

**What went wrong:** ORB-SLAM3 repeatedly failed to initialize:
```
Not enough motion for initializing. Resetting...
bad imu flag
IMU initialization failed. Resetting active map...
```

**Diagnosis:** Step 1 — IMU noise parameters looked reasonable. Step 2 — IMU frequency (400Hz) matched the bag data. Step 3 — I examined the `IMU.T_b_c1` extrinsic matrix and realized the AI had copied a LiDAR-to-Camera transform, not IMU-to-Camera. The two sensors are at different positions on the drone, so the geometric relationship was completely wrong.

**Decision:** We chose to focus on pure Monocular mode rather than spend uncertain time calibrating the correct extrinsic. This turned out well — our optimized Monocular v2 achieved ATE=10.65m (88% improvement over baseline).

**Lessons:** A working simple solution beats a broken complex one. The AI pattern-matched from the calibration file without understanding which sensor the matrix belonged to — I had to identify the root cause by thinking about the physical sensor layout myself.

---

## Section 5: Learning Growth

**Skills improved:**
- **Docker**: Before this course, I had almost no experience with Docker. Now I can create containers, mount volumes, copy files, and manage processes inside containers comfortably.
- **ROS**: I learned how topics, subscribers, and publishers work, how to remap topics for rosbag playback, and how to coordinate multiple ROS nodes (roscore, SLAM node, rosbag).
- **SLAM concepts**: I now understand ORB features, FAST detection thresholds, Sim(3) alignment, scale ambiguity in monocular mode, and the difference between monocular and monocular-inertial SLAM.
- **Parameter tuning**: I learned to change one variable at a time, measure the effect, and analyze trade-offs between metrics.

**Confidence change:** At the start, running ORB-SLAM3 in Docker felt overwhelming — I did not even know how to read a YAML config file. Now I can set up the full pipeline from scratch and understand what each parameter does.

---

## Section 6: Critical Reflection

**AI both improved and hindered learning.** It accelerated development enormously — tasks that might take days (like writing the CLAHE code, creating evaluation scripts, or finding the System.cc bug) were done in minutes. Without it, I probably could not have finished the project on time. However, I sometimes accepted outputs without fully understanding them. For example, the AI generated the Umeyama alignment code for trajectory figures, and I did not immediately understand the math behind Sim(3) alignment. I had to go back and study it separately.

**Reliance level:** The AI wrote most of the code, which I recognize is a lot of reliance. But I always checked the output results carefully and questioned numbers that looked wrong (like 100.69% completeness). For the conceptual understanding of SLAM, evaluation metrics, and sensor calibration, I made sure to study these topics myself rather than just trusting the AI's explanations.

**Next time:** I would try to write simpler code myself first before asking the AI, especially for tasks like Python scripts. I would also ask the AI to explain its code line by line more often, rather than just running it and checking if the output looks right.

---

## Section 7: Evidence

### 7.1 Code Snippets

**System.cc patch:**
```cpp
// Original (line 572):
if(mSensor==MONOCULAR) { cerr << "ERROR: SaveTrajectoryTUM cannot be used for monocular."; return; }
// Patched:
if(false && mSensor==MONOCULAR)
```

**CLAHE preprocessing in ros_mono_compressed.cc:**
```cpp
if(mbClahe) {
    cv::Mat gray;
    cv::cvtColor(image, gray, cv::COLOR_BGR2GRAY);
    mClahe->apply(gray, gray);
    cv::cvtColor(gray, image, cv::COLOR_GRAY2BGR);
}
```

### 7.2 Error Logs

```
Not enough motion for initializing. Resetting...
bad imu flag
```
```
ERROR: SaveTrajectoryTUM cannot be used for monocular.
```
```
The token '&&' is not a valid statement separator in this version.
```

### 7.3 Prompt Examples

**Helpful:** *"The v1 ATE is 17.15m, Completeness only 90%. How to improve?"* — AI suggested 5000 features, FAST 8/3, 0.75x rate. This became v2 (ATE=10.65m, Completeness=94.51%).

**Misleading:** *"Set up Mono-Inertial config."* — AI used LiDAR extrinsic instead of IMU extrinsic. Mode failed to initialize.

**Helpful:** *"CameraTrajectory.txt not generated. Why?"* — AI found System.cc restriction in minutes and suggested the bypass patch.
