# StairMaster Step Counter

AI-powered StairMaster step counting from video using **pose estimation** and **signal processing**.

The project analyzes a video of a person using a StairMaster, extracts body keypoints, and estimates the number of steps using multiple counting algorithms.

---

##  Features

- Pose estimation using:
  - MediaPipe Pose
  - YOLOv8 Pose
- Six step-counting approaches:
  - Ankle Peak Detection
  - Ankle Zero-Crossing
  - Autocorrelation
  - Knee-Angle Peak Detection
  - Knee-Angle Zero-Crossing
  - Near-Leg Detection + Zero-Crossing
- Signal smoothing to reduce pose-estimation noise
- Visualization of movement signals and detected steps
- Comparison of different pose-estimation models and counting methods

---

##  How It Works

The complete pipeline is:

```text
Video
  ↓
Pose Estimation
  ↓
Hip / Knee / Ankle Keypoints
  ↓
Movement Signal
  ↓
Signal Smoothing
  ↓
Step Detection
  ↓
Step Count
````

For every frame, the pose-estimation model extracts body landmarks. The project mainly uses the **ankle, knee, and hip** landmarks to describe the movement of each leg.

Two types of movement signals are used:

- **Ankle vertical position (****`ankle_y`****)** 
- **Knee joint angle** 

The signals are smoothed before applying the step-counting algorithms.

---

# 🔬 Step Counting Algorithms

The project implements six different approaches. Each method uses a different way of detecting the periodic movement produced by StairMaster steps.

---

## 1. Ankle Peak Detection

This method tracks the **vertical position of the ankle** over time.

As the person steps, the ankle moves periodically:


```
Ankle Y
  ↑
  │       ●           ●
  │      / \         / \
  │_____/   \_______/   \____
  │
  └──────────────────────────→ Time
        Step          Step
```

The algorithm detects **local maxima (peaks)** in the ankle signal.

A minimum distance between peaks and a prominence threshold are applied to prevent small movements or noise from being counted as steps.

**In simple terms:**

> One significant ankle peak = one detected step.

---

## 2. Ankle Zero-Crossing

This method also uses the **ankle vertical position**, but instead of searching for peaks, it detects when the signal crosses its mean.

First, the signal is centered:


```
centered_signal = ankle_y - mean(ankle_y)
```

The algorithm then detects **upward crossings of zero**.


```
Signal
  ↑
  │       /\          /\
  │      /  \        /  \
──┼─────●────\──────●────\── Mean
  │    /      \    /      \
  │___/        \__/        \__
  └──────────────────────────→ Time
       ↑              ↑
    Crossing       Crossing
```

A minimum time interval is used to avoid counting noise as multiple steps.

**In simple terms:**

> Each valid upward crossing of the mean = one detected step.

This approach can be useful when the peaks are noisy or distorted.

---

## 3. Autocorrelation

Autocorrelation does not try to detect every individual step.

Instead, it analyzes the **entire ankle movement signal** to determine how often the movement repeats.

The signal is compared with shifted versions of itself:

```math
R(k) = \sum_t x(t)x(t+k)
```

where `k` represents the time lag.

For periodic StairMaster movement, the autocorrelation produces a strong response at the dominant step period.


```
Movement:

    /\      /\      /\
   /  \    /  \    /  \
__/    \__/    \__/    \__

          ↓

Autocorrelation:

        ●
       / \
______/   \____________
       ↑
  dominant period
```

Once the dominant period is estimated, the number of steps is calculated from the duration of the analyzed signal.

For example:


```
Step period = 2 seconds
Signal duration = 20 seconds

Estimated steps = 20 / 2 = 10
```

**In simple terms:**

> Find how often the movement repeats, then estimate the number of repetitions.

---

## 4. Knee-Angle Peak Detection

Instead of tracking the ankle position, this method analyzes the **angle of the leg**.

The angle is calculated using:


```
Hip → Knee → Ankle
       Hip
        ●
        |
        |
        ● Knee
       /
      /
     ● Ankle
```

For every frame, the knee angle is calculated, producing a temporal knee-angle signal.

The algorithm then detects **local peaks** in this signal.

**In simple terms:**

> One significant knee-angle peak = one detected step.

This provides an alternative to ankle tracking by using the geometry of the entire leg.

---

## 5. Knee-Angle Zero-Crossing

This method uses the **knee-angle signal**, but applies zero-crossing detection instead of peak detection.

The knee-angle signal is centered around its mean:


```
centered_angle = knee_angle - mean(knee_angle)
```

The algorithm then detects **upward crossings** of the mean.


```
Knee Angle
  ↑
  │       /\          /\
  │      /  \        /  \
──┼─────●────\──────●────\── Mean
  │    /      \    /      \
  │___/        \__/        \__
  └──────────────────────────→ Time
       ↑              ↑
    Crossing       Crossing
```

**In simple terms:**

> Each valid upward crossing of the knee-angle signal = one detected step.

This combines the geometric information of the knee angle with the zero-crossing approach.

---

## 6. Near-Leg Detection + Zero-Crossing

This method is designed to handle **leg occlusion**.

In the StairMaster video, one leg may partially hide the other from the camera. Instead of relying equally on both legs, the system uses the **pose-estimation confidence** to determine which leg is tracked more reliably.

For example:


```
Left leg confidence   = 0.91
Right leg confidence  = 0.47

          ↓

Selected leg = Left
```

The leg with the higher average confidence is selected as the **near/reliable leg**.

Zero-crossing detection is then applied only to that leg.

Because the StairMaster pedals are mechanically linked, the system assumes that the other leg performs a corresponding number of movements.

Therefore:

```math
Total\ Steps =
Detected\ Steps_{reliable\ leg} \times 2
```

For example:


```
Reliable leg = 8 detected steps

Total = 8 × 2 = 16 steps
```

**In simple terms:**

> Select the leg tracked with the highest confidence, count its movements, then multiply by 2.

This approach is specifically intended to reduce errors caused by one leg being occluded or poorly tracked.

---

##  Algorithm Comparison

| Algorithm | Signal | Detection Method | Main Purpose |
|---|---|---|---|
| **Ankle Peak Detection** | Ankle Y | Local maxima | Detect steps from ankle peaks |
| **Ankle Zero-Crossing** | Ankle Y | Mean crossings | Handle distorted/noisy peaks |
| **Autocorrelation** | Ankle Y | Dominant period | Detect periodic movement |
| **Knee-Angle Peak Detection** | Knee Angle | Local maxima | Detect steps from leg geometry |
| **Knee-Angle Zero-Crossing** | Knee Angle | Mean crossings | Alternative knee-based detection |
| **Near-Leg + Zero-Crossing** | Reliable leg | Mean crossings × 2 | Handle leg occlusion |

---

##  Pose Estimation

Two pose-estimation backends are evaluated:

### MediaPipe Pose

MediaPipe Pose extracts human body landmarks from each frame and provides confidence values for the detected landmarks.

### YOLOv8 Pose

YOLOv8 Pose is used as an alternative pose-estimation backend to compare its results with MediaPipe.

Both models provide the keypoints required to construct the ankle and knee movement signals.

---

##  Signal Processing

Pose estimation can produce noisy coordinates because of:

-  Small tracking errors 
-  Occlusion 
-  Motion blur 
-  Changes in detection confidence 

The extracted signals are therefore smoothed before step detection.

This reduces small frame-to-frame fluctuations while preserving the main periodic stepping pattern.

---

##  Project Structure


```
StepCounter/
│
├── stairmaster_step_counter_multi_algo.ipynb
├── requirements_windowed.txt
├── .gitignore
└── README.md
```

Video files and model weights are excluded from the repository through `.gitignore`.

---

##  Technologies

-  Python 
-  OpenCV 
-  NumPy 
-  SciPy 
-  Matplotlib 
-  MediaPipe 
-  Ultralytics YOLO 
-  Jupyter Notebook 

---

##  Installation

Clone the repository:


```
git clone https://github.com/yassine532/StepCounter.git
cd StepCounter
```

Create a virtual environment:


```
python -m venv stairmaster_env
source stairmaster_env/bin/activate
```

Install the dependencies:


```
pip install -r requirements_windowed.txt
```

---

##  Usage

Open the notebook:


```
jupyter notebook stairmaster_step_counter_multi_algo.ipynb
```

Then provide a StairMaster video and run the notebook.

The notebook will:

1.  Load the video. 
2.  Run pose estimation. 
3.  Extract hip, knee, and ankle landmarks. 
4.  Generate movement signals. 
5.  Smooth the signals. 
6.  Apply the six step-counting algorithms. 
7.  Compare the resulting step counts. 

---

##  Objective

The objective of this project is to evaluate different **computer vision and signal-processing techniques** for reliable StairMaster step counting from video.

The main challenges addressed are:

-  Pose-estimation noise 
-  Leg occlusion 
-  False peak detection 
-  Irregular movement 
-  Periodic motion 
-  Differences between pose-estimation models 

---

## Status

 **Work in progress**

Different pose-estimation models and signal-processing approaches are being evaluated to determine the most reliable configuration for StairMaster step counting.
