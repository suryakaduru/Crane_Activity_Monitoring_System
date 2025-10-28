
# Complete Documentation: Crane Load Detection System

## Table of Contents

1. [System Overview](#system-overview)
2. [Configuration Parameters Explained](#configuration-parameters-explained)
3. [System Initialization](#system-initialization)
4. [Core Modules and Methods](#core-modules-and-methods)
5. [State Transition Logic](#state-transition-logic)
6. [Complete Processing Flow](#complete-processing-flow)
7. [Troubleshooting Guide](#troubleshooting-guide)

***

## 1. System Overview

### What Does This System Do?

This system monitors crane operations in industrial settings using video analysis. It automatically detects and classifies three key activities:

- **Load Lifting**: When the crane picks up a load and lifts it upward
- **Moving Load**: When the crane is actively moving with a load
- **Unloading**: When the crane sets down the load after movement
- **Idle/Static**: When the crane is not performing any active operation


### How Does It Work?

The system uses two main technologies:

1. **YOLO Object Detection**: Finds the crane hook in each video frame
2. **Motion Analysis**: Tracks how the hook and load move between frames

***

## 2. Configuration Parameters Explained

### Section: [PARAMS]

| Parameter | Default Value | What It Does | Effect of Increasing | Effect of Decreasing |
| :-- | :-- | :-- | :-- | :-- |
| **static_threshold** | 0.7 | Threshold below which motion is considered "static" | System becomes stricter—requires less motion to call it "static" | System becomes lenient—allows more motion while still calling it "static" |
| **movement_threshold** | 0.65 | Threshold above which motion is considered "moving load" | Requires stronger motion to detect "moving load" | Detects "moving load" even with weaker motion |
| **window_size** | 15 | Number of frames used to smooth motion detection | More smoothing, slower to react to changes | Less smoothing, faster reaction but more flickering |
| **smoothing_factor** | 0.4 | Not actively used in current code | N/A | N/A |
| **min_features** | 80 | Minimum number of trackable points needed | Requires more features, more strict tracking | Requires fewer features, works in simpler scenes |

### Optical Flow Parameters

| Parameter | Value | What It Does |
| :-- | :-- | :-- |
| **maxCorners** | 200 | Maximum number of corner points to track in the image |
| **qualityLevel** | 0.01 | Minimum quality of corners (0 to 1 scale) |
| **minDistance** | 7 | Minimum distance between detected corners (pixels) |
| **blockSize** | 7 | Size of averaging block for corner detection |

**Lucas-Kanade Parameters:**


| Parameter | Value | What It Does |
| :-- | :-- | :-- |
| **winSize** | (15,15) | Size of search window at each pyramid level |
| **maxLevel** | 2 | Maximum pyramid levels (0 means no pyramid) |
| **criteria** | (3,10,0.03) | Termination criteria for iterative search |

### Vertical Movement Detection

| Parameter | Default Value | What It Does | Effect of Changing |
| :-- | :-- | :-- | :-- |
| **vert_window** | 25 | Number of frames to analyze for vertical trend | Higher = smoother vertical detection but slower response |
| **vert_px_threshold** | 0.005 | Fraction of ROI height for minimum vertical movement | Higher = needs more movement to detect lifting/lowering |
| **steady_up_count** | 3 | Confirmations needed before calling movement "upward" | Higher = more certain but slower detection |
| **unloading_static_time** | 3.0 | Seconds of stillness required before confirming unloading | Higher = takes longer to confirm unloading |

### Section: [YOLO]

| Parameter | Value | What It Does |
| :-- | :-- | :-- |
| **model_path** | hook_oct14.pt | Path to trained YOLO model file for hook detection |
| **confidence** | 0.5 | Minimum confidence score (0-1) for accepting detections |

### Section: [INPUT1]

| Parameter | Value | What It Does |
| :-- | :-- | :-- |
| **type** | rtsp | Type of video input (rtsp for cameras, file for videos) |
| **path** | /home/.../concrete_bucket.mp4 | Full path to video file or RTSP stream URL |


***

## 3. System Initialization

### Step-by-Step Initialization Process

#### Step 1: Configuration Loading

```
Program starts → Reads config.ini → Loads all parameters into memory
```

**What happens:**

- The program reads the configuration file
- All threshold values, file paths, and parameters are stored
- If any parameter is missing, default fallback values are used


#### Step 2: Video Capture Setup

```
Open video file → Get video properties (FPS, width, height)
```

**What happens:**

- Opens the video file specified in config
- Retrieves frame rate (e.g., 25 FPS)
- Retrieves frame dimensions (e.g., 1920x1080 pixels)
- Initializes video writer for output


#### Step 3: YOLO Thread Initialization

```
Load YOLO model → Start detection thread → Thread runs in background
```

**What happens:**

- Loads the trained YOLO model weights
- Creates a separate processing thread
- Thread waits for frames to detect crane hooks
- Runs independently without blocking main processing


#### Step 4: CraneLoadDetector Initialization

**Memory Structures Created:**


| Structure | Purpose | Initial State |
| :-- | :-- | :-- |
| `of_window` | Stores last 15 optical flow values | Empty queue |
| `fd_window` | Stores last 15 frame difference values | Empty queue |
| `center_history` | Stores last 25 vertical positions | Empty queue |
| `state_history` | Stores last 75 motion decisions | Empty queue |
| `unloading_y_buffer` | Temporary buffer for unloading detection | Not created yet |
| `previous_frame` | Previous frame for comparison | None |
| `previous_features` | Tracked feature points | None |
| `lifting` | Is load being lifted? | False |
| `current_state` | Current crane state | "STATIC" |


***

## 4. Core Modules and Methods

### Module 1: Frame Preprocessing

**Method:** `preprocess(frame)`

**Input:** Color video frame (BGR format)

**Process:**

```
Color Frame → Convert to Grayscale → Apply Gaussian Blur → Return Processed Frame
```

**Step-by-step:**

1. Converts BGR color frame to single-channel grayscale
2. Applies 5x5 Gaussian blur to reduce noise
3. Returns smoothed grayscale image

**Why this matters:**

- Grayscale reduces data (3 channels → 1 channel)
- Blur removes small noise that could confuse tracking
- Makes motion detection more reliable

***

### Module 2: Feature Detection

**Method:** `detect_features(gray)`

**Input:** Grayscale frame

**Process:**

```
Grayscale Image → Find Corners (Shi-Tomasi) → Return Feature Points
```

**What it does:**

- Uses Shi-Tomasi corner detection algorithm
- Finds up to 200 "interesting" points in the image
- These points are edges, corners, textures—things easy to track
- Returns coordinates of these points

**Example:**
If analyzing a crane hook image, it might find:

- Corners of the hook
- Texture patterns on the load
- Edge transitions

***

### Module 3: Feature Detection in Patch

**Method:** `detect_features_in_patch(gray, center)`

**Input:** Grayscale frame + center point coordinates

**Process:**

```
1. Calculate patch boundaries around center (80x80 pixels)
2. Extract that small region
3. Find features only in that region
4. Map coordinates back to full frame
5. Return feature points
```

**Why use this:**

- More efficient—only searches small area
- Focuses on the detected hook location
- Reduces false tracking from background motion

***

### Module 4: Optical Flow Calculation

**Method:** `calc_optical_flow(prev_gray, curr_gray)`

**Input:** Previous frame + Current frame (both grayscale)

**Output:** Average motion magnitude, Average angle

**Detailed Process:**


| Step | What Happens | Example |
| :-- | :-- | :-- |
| 1. Check features | Do we have trackable points from previous frame? | Yes: 150 points available |
| 2. Track features | Use Lucas-Kanade to find where each point moved | Point at (100,200) moved to (102,198) |
| 3. Filter results | Keep only successfully tracked points | 140 out of 150 tracked successfully |
| 4. Calculate displacement | For each point: how far did it move? | Point moved 2.8 pixels |
| 5. Calculate angle | What direction did points move? | Moved at -45 degrees (up-left) |
| 6. Average values | Average all displacements and angles | Avg: 2.1 pixels, -30 degrees |

**Real-world interpretation:**

- **High magnitude (e.g., 5+ pixels)**: Significant motion, crane likely moving
- **Low magnitude (e.g., <1 pixel)**: Little motion, crane likely static
- **Negative angle with upward motion**: Load being lifted
- **Positive angle with downward motion**: Load being lowered

***

### Module 5: Frame Difference Calculation

**Method:** `calc_frame_diff(prev_gray, curr_gray)`

**Input:** Previous frame + Current frame

**Output:** Percentage of frame that changed

**Detailed Process:**


| Step | Action | Technical Detail |
| :-- | :-- | :-- |
| 1. Subtract frames | Calculate absolute difference | `diff = |current - previous|` |
| 2. Threshold | Convert to binary (changed/not changed) | Pixels with diff > 25 = changed |
| 3. Find contours | Detect regions of change | Connected components of white pixels |
| 4. Measure area | Sum up all changed regions | Total area of all contours |
| 5. Calculate percentage | Divide by total frame area × 100 | (changed_area / total_area) × 100 |

**Example:**

- Frame size: 1920 × 1080 = 2,073,600 pixels
- Changed area: 50,000 pixels
- Frame difference: (50,000 / 2,073,600) × 100 = **2.4%**

**Interpretation:**

- **< 0.5%**: Almost no change, very static
- **0.5% - 2%**: Moderate change, possible motion
- **> 2%**: Significant change, definite motion

***

### Module 6: Motion Classification

**Method:** `classify_motion(of_avg, fd_avg, ang)`

**Input:**

- `of_avg`: Average optical flow magnitude
- `fd_avg`: Average frame difference percentage
- `ang`: Average movement angle

**Output:** Motion state ("STATIC" or "MOVING_LOAD")

**Detailed Algorithm:**

```
Step 1: Calculate Combined Motion Metric
-----------------------------------------
motion_metric = (0.8 × optical_flow) + (0.4 × frame_difference)

Example:
  optical_flow = 3.2 pixels
  frame_difference = 1.8%
  motion_metric = (0.8 × 3.2) + (0.4 × 1.8) = 2.56 + 0.72 = 3.28

Step 2: Apply Angle Factor
---------------------------
If angle > 5 degrees: multiply by 1.0 (full weight)
If angle ≤ 5 degrees: multiply by 0.5 (reduce weight)

Example: angle = 12 degrees
  motion_metric = 3.28 × 1.0 = 3.28

Step 3: Determine Frame-Level Motion
------------------------------------
If optical_flow < 0.3 OR motion_metric < movement_threshold:
  → current_frame_moving = False
Else:
  → current_frame_moving = True

Example: motion_metric = 3.28, movement_threshold = 0.65
  → current_frame_moving = True

Step 4: Update State History
-----------------------------
Add current_frame_moving to history (keeps last 75 frames)
History = [False, False, True, True, True, ... True]

Step 5: Update Counters
-----------------------
If current_frame_moving = True:
  moving_counter += 1
  static_counter = 0
Else:
  static_counter += 1
  moving_counter = 0

Step 6: Majority Voting
-----------------------
Count how many "True" values in last 75 frames
moving_count = sum of True values

If moving_count ≥ 50 AND moving_counter ≥ 8:
  → State = "MOVING_LOAD"
Else if (75 - moving_count) ≥ 50 AND static_counter ≥ 60:
  → State = "STATIC"
Else:
  → State = previous_state (no change)
```

**State Transition Table:**


| Current State | Condition | New State | Frames Required |
| :-- | :-- | :-- | :-- |
| STATIC | 50+ of last 75 frames show motion + 8+ consecutive moving | MOVING_LOAD | ~50-60 frames |
| MOVING_LOAD | 50+ of last 75 frames show no motion + 60+ consecutive static | STATIC | ~60-75 frames |
| Any | Unclear (between thresholds) | Keep previous state | N/A |


***

### Module 7: Vertical Trend Detection

**Method:** `detect_vertical_trend()`

**Purpose:** Determine if load is moving up, down, or staying steady

**Detailed Process:**

```
Step 1: Check History Buffer
-----------------------------
center_history contains last 25 Y-coordinates of detected load
If less than 25 values available → return "none"

Step 2: Calculate Vertical Displacement
---------------------------------------
dy = Y_position_now - Y_position_25_frames_ago

Example:
  Y now (frame 100) = 450 pixels
  Y then (frame 75) = 480 pixels
  dy = 450 - 480 = -30 pixels (negative = moved UP)

Step 3: Calculate Thresholds
----------------------------
threshold_high = max(0.005 × ROI_height, 1.2)
threshold_low = max(0.005 × ROI_height, 0.4)

Example with ROI_height = 1080:
  threshold_high = max(0.005 × 1080, 1.2) = max(5.4, 1.2) = 5.4 pixels
  threshold_low = max(0.005 × 1080, 0.4) = max(5.4, 0.4) = 5.4 pixels

Step 4: Determine Instantaneous Trend
-------------------------------------
If |dy| < threshold_low:
  → trend = "steady"
  → steady_counter += 1
Else if |dy| > threshold_high:
  If dy < 0: → trend = "up"
  If dy > 0: → trend = "down"
  → steady_counter = 0
Else:
  → trend = previous trend (keep last decision)

Step 5: Update Trend History
----------------------------
Add trend to history (keeps last 6 trends)
trend_history = ["steady", "steady", "up", "up", "up", "up"]

Step 6: Majority Confirmation
-----------------------------
Count occurrences: {"up": 4, "steady": 2}
Most common = "up"

If "up" or "down" appears ≥ 3 times:
  → Confirmed trend = that direction
Else if steady_counter ≥ 20 frames:
  → Confirmed trend = "steady"
Else:
  → Keep last confirmed trend
```

**Vertical Trend Decision Table:**

| dy Value | Absolute |dy| | Interpretation | Confirmed Trend After |
|----------|-----------------|----------------|----------------------|
| -30 pixels | 30 pixels | Strong upward | 3-6 frames of "up" |
| -2 pixels | 2 pixels | Weak upward or steady | 6+ frames |
| 0 pixels | 0 pixels | No movement | 20 frames of steady |
| +2 pixels | 2 pixels | Weak downward or steady | 6+ frames |
| +25 pixels | 25 pixels | Strong downward | 3-6 frames of "down" |

***

### Module 8: Lifting Detection Logic

**Purpose:** Determine if crane is actively lifting a load

**Complex State Machine:**

```
State Variables:
- lifting: Boolean (Is lifting happening?)
- lifting_freeze: Boolean (Has lifting been confirmed and frozen?)
- lifting_freeze_start: Timestamp when freeze started
- lift_false_counter: Counter for non-lifting frames
- false_pending: Boolean (Waiting to confirm lifting should stop?)
- false_pending_start: Timestamp when false pending started

Lifting Detection Flow:
=======================

CASE 1: Unloading Detected
---------------------------
If unloading = True:
  → lifting = False (immediately)
  → lifting_freeze = False
  → Reset all counters
  → EXIT

CASE 2: Immediate Lift Conditions Met
--------------------------------------
immediate_lift = True when:
  - (vertical_trend = "up" AND motion_state = "STATIC") 
    OR
  - motion_state = "MOVING_LOAD"

Example: Load detected, moving upward but crane body static
  → immediate_lift = True

Sub-case 2A: Freeze Not Yet Confirmed
--------------------------------------
If lifting_freeze = False:
  If this is first immediate_lift frame:
    → Start timer: lifting_freeze_start = now
  Else if timer running > 1.0 seconds:
    → lifting_freeze = True (FREEZE ACTIVATED)
  
  If lifting_freeze just became True:
    → lifting = True
    → Reset all false counters

Sub-case 2B: Already Frozen
----------------------------
If lifting_freeze = True:
  → lifting = True (maintain)

CASE 3: Immediate Lift Conditions NOT Met
-----------------------------------------
If lifting_freeze = True:
  → lifting = True (keep True even without immediate lift)
  → Freeze prevents false negatives during brief pauses

Else (lifting_freeze = False):
  → lift_false_counter += 1
  
  If lift_false_counter ≥ 600 frames (~24 seconds at 25fps):
    If not false_pending:
      → false_pending = True
      → false_pending_start = now
    
    Else if time since false_pending_start > 10 seconds:
      → lifting = False (FINALLY TURN OFF)
      → Reset all states
  
  Else (counter < 600):
    → false_pending = False
  
  → lifting_freeze_start = None (reset freeze timer)
```

**Lifting State Transition Diagram:**


| Current Lifting State | Condition | Action | New Lifting State | Time Required |
| :-- | :-- | :-- | :-- | :-- |
| False | immediate_lift = True | Start freeze timer | False (waiting) | 0 sec |
| False (waiting) | Timer > 1 sec | Activate freeze | True | 1 sec |
| True (frozen) | immediate_lift = False | Start false counter | True (maintained) | 0 sec |
| True (frozen) | false_counter > 600 | Start false pending | True (maintained) | 24 sec |
| True (false pending) | false_pending > 10 sec | Turn off lifting | False | 10 sec |
| True | Unloading detected | Immediate reset | False | Instant |

**Why This Complexity?**

- **1-second freeze**: Prevents flickering from brief detection losses
- **600-frame counter**: Requires sustained absence before considering turning off
- **10-second pending**: Additional safety period before confirming lifting stopped
- **Result:** Stable, reliable lifting detection without constant on/off flickering

***

### Module 9: Unloading Detection Logic

**Purpose:** Detect when crane sets down load after movement

**Detailed Algorithm:**

```
Trigger Condition:
==================
Unloading detection activates when:
  - motion_state = "STATIC" (crane stopped moving)
  AND
  - previous_state = "MOVING_LOAD" (was moving just before)
  OR
  - unloading_y_buffer already exists (in middle of unloading check)

Initialization (First Frame After Motion Stops):
================================================
Create new temporary structures:
  - unloading_y_buffer = [] (empty list for Y positions)
  - static_start_time = current time
  - missing_frames = 0
  - ema_y = None (exponential moving average)

Frame-by-Frame Processing:
===========================

Step 1: Get Current Y Position
-------------------------------
Priority order:
  1. If 5+ feature points tracked:
     → y_val_raw = median Y of all feature points
  2. Else if bounding box detected:
     → y_val_raw = center Y of bounding box
  3. Else:
     → y_val_raw = last_known_y (use previous value)

Example: 20 features with Y values [445, 447, 448, 450, 451, ...]
  → y_val_raw = median = 449 pixels

Step 2: Exponential Moving Average Smoothing
--------------------------------------------
Purpose: Reduce noise in Y position measurements

alpha = 0.4 (smoothing factor)

If first value:
  ema_y = y_val_raw
Else:
  ema_y = (0.4 × y_val_raw) + (0.6 × previous_ema_y)

Example:
  y_val_raw = 449
  previous_ema_y = 450
  ema_y = (0.4 × 449) + (0.6 × 450) = 179.6 + 270 = 449.6

Step 3: Update Buffer
---------------------
Add ema_y to unloading_y_buffer
Buffer now: [450.2, 450.0, 449.8, 449.6]

Update tracking:
  last_known_y = ema_y
  missing_frames = 0

Step 4: Check Unloading Criteria
--------------------------------
window_size = 10 frames
tolerance = 8.0 pixels
min_time_static = 45 seconds

Requirements ALL must be met:
  1. Time since static_start_time > 45 seconds
  2. Buffer has ≥ 10 values
  3. Variation in last 10 values ≤ 8 pixels
  4. Missing frames < 3

Calculation:
  recent_10 = last 10 values from buffer
  max_y = 451.2
  min_y = 448.8
  variation = 451.2 - 448.8 = 2.4 pixels

If variation ≤ 8.0 AND time > 45 sec:
  → unloading = True
  → Clear buffer and reset

Cleanup:
========
If motion_state changes (not STATIC anymore):
  → Delete unloading_y_buffer
  → Reset all temporary variables
```

**Unloading Detection Timeline:**


| Time | Event | Buffer Status | Decision |
| :-- | :-- | :-- | :-- |
| T+0s | Motion stops | Empty → Start collecting | Not unloading |
| T+1s | Still static | 25 values | Not unloading (time < 45s) |
| T+10s | Still static | 250 values | Not unloading (time < 45s) |
| T+30s | Still static | 750 values | Not unloading (time < 45s) |
| T+45s | Still static, Y stable | 1125 values, variation = 3.2px | **UNLOADING = TRUE** |
| T+46s | - | Buffer cleared | Reset, waiting |

**Why 45 Seconds?**

- Industrial crane operations: unloading takes time
- Workers need to unhook, secure load
- Prevents false positives from brief pauses during transit
- Ensures actual unloading, not just temporary stop

***

## 5. State Transition Logic

### Complete State Diagram

```
┌───────────────────────────────────────────────────┐
│                  SYSTEM START                     │
│              Initial State: STATIC                │
└──────────────────────┬────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────┐
         │                         │
         │         STATIC          │/home/quantic/Datasets/LNT_Crane/test_videos/wooden_bar.mp4
         │   (Crane Idle/Stopped)  │
         │                         │
         └─────────────────────────┘
                │           ▲
                │           │
     Motion     │           │  No motion
     detected   │           │  50+ frames
     50+ frames │           │  60+ consecutive
     8+ consec  │           │
                │           │
                ▼           │
         ┌─────────────────────────┐
         │                         │
         │     MOVING_LOAD         │
         │  (Crane Transporting)   │
         │                         │
         └─────────────────────────┘
                │           ▲
                │           │
     Stops      │           │  Resumes
     moving     │           │  motion
                │           │
                ▼           │
         ┌─────────────────────────┐
         │                         │
         │   STATIC (checking)     │
         │   + Unloading Buffer    │
         │                         │
         └─────────────────────────┘
                │           ▲
                │           │
     45 sec     │           │  Variation
     stable     │           │  > 8 pixels
     variation  │           │  OR
     ≤ 8 pixels │           │  motion resumes
                │           │
                ▼           │
         ┌─────────────────────────┐
         │                         │
         │      UNLOADING          │
         │   (Setting Down Load)   │
         │                         │
         └─────────────────────────┘
                │
                │ Unloading
                │ complete
                ▼
         ┌────────────────────────┐
         │                        │
         │        STATIC          │
         │      (Ready for        │
         │      next cycle)       │
         │                        │
         └────────────────────────┘
```


### Detailed State Transition Table

| From State | To State | Trigger Conditions | Approximate Time | Additional Checks |
| :-- | :-- | :-- | :-- | :-- |
| **STATIC** | **MOVING_LOAD** | -  50+ of last 75 frames show motion<br>-  8+ consecutive moving frames<br>-  motion_metric > 0.65 | 2-3 seconds | Optical flow > 0.3 pixels |
| **MOVING_LOAD** | **STATIC** | -  50+ of last 75 frames show no motion<br>-  60+ consecutive static frames<br>-  motion_metric < 0.65 | 2.5-3 seconds | Begins unloading check |
| **STATIC (checking)** | **UNLOADING** | -  Static for 45+ seconds<br>-  Y-position variation ≤ 8 pixels<br>-  Missing frames < 3 | 45 seconds | Buffer has 10+ values |
| **UNLOADING** | **STATIC** | -  Unloading confirmed and logged<br>-  Buffer cleared | Instant | Reset all states |
| **MOVING_LOAD** | **MOVING_LOAD** | -  Motion continues | Continuous | Maintain state |
| **STATIC (checking)** | **STATIC** or **MOVING** | -  Motion resumes OR<br>-  Y variation > 8 pixels | Instant | Cancel unloading check, clear buffer |

### Lifting Flag Transitions (Independent of Main State)

| Lifting Status | Trigger | Time to Activate | Time to Deactivate |
| :-- | :-- | :-- | :-- |
| **False → True** | Vertical trend = "up" for 1+ second | 1 second | N/A |
| **True → True (maintained)** | Freeze active, even if conditions change | Instant | N/A |
| **True → False** | No immediate_lift for 24 seconds + 10 second confirmation | 34 seconds total | Instant upon confirmation |
| **True → False (forced)** | Unloading detected | N/A | Instant |


***

## 6. Complete Processing Flow

### Frame-by-Frame Processing Sequence

```
┌─────────────────────────────────────────────────────────────┐
│  FRAME N arrives from video                                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 1: ROI Extraction and Padding                         │
│  ───────────────────────────────────                        │
│  • Extract ROI: frame[0:1080, 956:1144]                     │
│  • ROI size: 188×1080 pixels                                │
│  • Pad to square: 1080×1080 (add 446px left, 446px right)   │
│  • Result: Padded frame ready for YOLO                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 2: YOLO Detection (Parallel Thread)                   │
│  ─────────────────────────────────────                      │
│  • YOLO thread receives padded frame                        │
│  • Runs object detection for crane load                     │
│  • Filters detections by confidence > 0.5                   │
│  • Returns bounding boxes in padded frame coordinates       │
│  • Example: [(540, 300, 580, 360)]                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 3: Coordinate Remapping                               │
│  ────────────────────────────                               │
│  • Convert padded coordinates back to original frame        │
│  • x_original = x_padded - 446 + 956                        │
│  • y_original = y_padded + 0                                │
│  • Example: (540-446+956, 300) = (1050, 300)                │
│  • Bounding boxes now in full frame coordinates             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 4: Full Frame Preprocessing                           │
│  ───────────────────────────────                            │
│  • Convert full frame to grayscale                          │
│  • Apply Gaussian blur (5×5 kernel)                         │
│  • Result: Smoothed grayscale frame                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 5: Optical Flow Calculation (if not first frame)      │
│  ──────────────────────────────────────────────────         │
│  • Use previous frame's features                            │
│  • Track features using Lucas-Kanade                        │
│  • Calculate displacement vectors                           │
│  • Compute average magnitude: e.g., 2.3 pixels              │
│  • Compute average angle: e.g., -15 degrees                 │
│  • Add to of_window (keeps last 15 values)                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 6: Frame Difference Calculation                       │
│  ────────────────────────────────────                       │
│  • Subtract: |current_frame - previous_frame|               │
│  • Threshold differences > 25                               │
│  • Find contours of changed regions                         │
│  • Calculate percentage: e.g., 1.8%                         │
│  • Add to fd_window (keeps last 15 values)                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 7: Motion Classification                              │
│  ─────────────────────────────                              │
│  • Calculate motion_metric = 0.8×OF + 0.4×FD                │
│  • Apply angle factor                                       │
│  • Determine current_frame_moving (True/False)              │
│  • Add to state_history (keeps last 75 decisions)           │
│  • Majority voting: count True values                       │
│  • Decide: "MOVING_LOAD" or "STATIC"                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 8: Main Center Calculation                            │
│  ───────────────────────────────                            │
│  • If detections exist:                                     │
│    - Find largest bounding box by area                      │
│    - Calculate center: cx = (x1+x2)/2, cy = (y1+y2)/2       │
│    - Add cy to center_history                               │
│    - Detect features in 80×80 patch around center           │
│    - Update previous_features for next frame                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 9: Vertical Trend Detection                           │
│  ────────────────────────────────                           │
│  • Compare Y positions over last 25 frames                  │
│  • Calculate dy = Y_now - Y_25_frames_ago                   │
│  • Determine trend: "up", "down", or "steady"               │
│  • Apply majority confirmation from last 6 trends           │
│  • Result: e.g., "up" (lifting detected)                    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 10: Lifting Detection Logic                           │
│  ────────────────────────────────                           │
│  • Check if unloading = True (if so, lifting = False)       │
│  • Calculate immediate_lift condition                       │
│  • Manage lifting_freeze state machine                      │
│  • Update lift_false_counter                                │
│  • Apply 1-second freeze before confirming lifting          │
│  • Apply 34-second delay before turning off lifting         │
│  • Result: lifting = True or False                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 11: Unloading Detection (if applicable)               │
│  ───────────────────────────────────────                    │
│  • Check if motion_state = "STATIC" after "MOVING_LOAD"     │
│  • If yes, create/update unloading_y_buffer                 │
│  • Apply exponential moving average to Y positions          │
│  • Check criteria: 45 sec + variation ≤ 8 pixels            │
│  • If met: unloading = True                                 │
│  • Else: continue buffering or reset if motion resumes      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 12: State Update and Logging                          │
│  ─────────────────────────────────                          │
│  • Determine new_state: "UNLOADING", "MOVING_LOAD", "STATIC"│
│  • If state changed from previous:                          │
│    - Log transition with timestamp                          │
│    - Record OF, FD, angle, trend values                     │
│  • Update current_state                                     │
│  • Update previous_state for next frame                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 13: Visualization                                     │
│  ──────────────────────                                     │
│  • Draw semi-transparent overlay box                        │
│  • Display "Load Lifting: True/False" (colored)             │
│  • Display "Crane status: Moving/Idle" (colored)            │
│  • Display "Unloading Status: True/False" (colored)         │
│  • Draw bounding boxes around detections                    │
│  • Draw yellow circle at main center                        │
│  • Draw green ROI rectangle                                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 14: Output                                            │
│  ───────────────                                            │
│  • Write annotated frame to output video file               │
│  • Display frame in window                                  │
│  • Store previous_frame = current grayscale                 │
│  • Wait for next frame                                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
                 Next Frame (N+1)
```


### Timing Analysis (at 25 FPS)

| Component | Processing Time | Notes |
| :-- | :-- | :-- |
| Frame capture | ~1 ms | Depends on video source |
| ROI extraction \& padding | <1 ms | Simple array operations |
| YOLO detection | 30-50 ms | Runs in parallel thread |
| Grayscale conversion | 2-3 ms | Fast OpenCV operation |
| Optical flow | 5-10 ms | Depends on feature count |
| Frame difference | 3-5 ms | Threshold and contour finding |
| Motion classification | <1 ms | Mathematical operations |
| Vertical trend | <1 ms | Queue operations |
| Lifting logic | <1 ms | State machine updates |
| Unloading check | 1-2 ms | Buffer management |
| Visualization | 5-8 ms | Drawing operations |
| **Total per frame** | **15-25 ms** | Well within 40ms budget (25 FPS) |


***

## 7. Troubleshooting Guide

### Common Issues and Solutions

| Problem | Possible Cause | Solution | Config Parameter to Adjust |
| :-- | :-- | :-- | :-- |
| **Lifting detected when crane is idle** | Threshold too sensitive | Increase `movement_threshold` | `movement_threshold = 0.7` to `0.8` |
| **Lifting not detected during actual lift** | Threshold too strict | Decrease `movement_threshold` | `movement_threshold = 0.65` to `0.5` |
| **State flickering between STATIC/MOVING** | Window size too small | Increase `window_size` | `window_size = 15` to `25` |
| **Unloading detected too quickly** | Time threshold too low | Increase `unloading_static_time` | `unloading_static_time = 3.0` to `5.0` or `10.0` |
| **Unloading never detected** | Position variation too strict | Increase tolerance in code | Modify `tolerance = 8.0` to `12.0` or `15.0` |
| **Vertical trend always "steady"** | Threshold too high | Decrease `vert_px_threshold` | `vert_px_threshold = 0.005` to `0.003` |
| **Too many false hook detections** | YOLO confidence too low | Increase `confidence` | `confidence = 0.5` to `0.6` or `0.7` |
| **Hook not detected** | YOLO confidence too high | Decrease `confidence` | `confidence = 0.5` to `0.4` or `0.3` |
| **Processing too slow** | Too many features tracked | Decrease `maxCorners` | `maxCorners = 200` to `100` |

### Parameter Tuning Guidelines

**For Faster Response (Less Stable):**

- Decrease `window_size` (15 → 10)
- Decrease required frame counts in code (50 → 30)
- Decrease `unloading_static_time` (3.0 → 1.5)

**For More Stable Detection (Slower Response):**

- Increase `window_size` (15 → 25)
- Increase required frame counts in code (50 → 70)
- Increase `unloading_static_time` (3.0 → 10.0)

**For Different Crane Speeds:**

- **Fast-moving cranes:** Decrease `movement_threshold` (0.65 → 0.5)
- **Slow-moving cranes:** Increase `movement_threshold` (0.65 → 0.8)

**For Different Camera Angles:**

- **Top-down view:** Decrease `vert_px_threshold` (more sensitive to small changes)
- **Side view:** Increase `vert_px_threshold` (larger pixel displacement for same movement)

