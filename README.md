# AI Real-time GYM Coach

> An AI-powered real-time workout assistant that combines computer vision, pose estimation, exercise analysis, LLM-generated coaching, text-to-speech, and workout history tracking.

[![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python\&logoColor=white)](https://www.python.org/)
[![Streamlit](https://img.shields.io/badge/Streamlit-1.54.0-FF4B4B?logo=streamlit\&logoColor=white)](https://streamlit.io/)
[![MediaPipe](https://img.shields.io/badge/MediaPipe-Pose_Landmarker-4285F4?logo=google\&logoColor=white)](https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker)
[![OpenCV](https://img.shields.io/badge/OpenCV-4.10.0-5C3EE8?logo=opencv\&logoColor=white)](https://opencv.org/)
[![Groq](https://img.shields.io/badge/Groq-LLM_API-F55036)](https://groq.com/)
[![SQLite](https://img.shields.io/badge/SQLite-Database-003B57?logo=sqlite\&logoColor=white)](https://www.sqlite.org/)

---

## Table of Contents

* [Overview](#overview)
* [Why This Project](#why-this-project)
* [Core Features](#core-features)
* [Supported Exercises](#supported-exercises)
* [How It Works](#how-it-works)
* [System Architecture](#system-architecture)
* [Computer Vision Pipeline](#computer-vision-pipeline)
* [Exercise Analysis](#exercise-analysis)
* [AI Coaching Pipeline](#ai-coaching-pipeline)
* [Workout Tracking and Persistence](#workout-tracking-and-persistence)
* [Project Structure](#project-structure)
* [Technology Stack](#technology-stack)
* [Requirements](#requirements)
* [Installation](#installation)
* [Environment Variables](#environment-variables)
* [Running the Application](#running-the-application)
* [Using the Application](#using-the-application)
* [Configuration](#configuration)
* [Deployment Notes](#deployment-notes)
* [Troubleshooting](#troubleshooting)
* [Limitations](#limitations)
* [Future Improvements](#future-improvements)
* [Contributing](#contributing)
* [License](#license)
* [Author](#author)

---

# Overview

**AI Real-time GYM Coach** is a browser-based fitness assistant built with Streamlit.

The application uses the device camera to analyze a person's body pose in real time. A MediaPipe Pose Landmarker model extracts pose landmarks, and an exercise-specific detector converts those landmarks into workout signals such as:

* Repetition count
* Joint angles
* Body alignment
* Movement stage
* Exercise depth
* Balance
* Swing detection
* Back arch
* Exercise-specific form status

These metrics are displayed live inside the Streamlit interface.

The project also includes an AI coaching layer. When a workout starts, a set is completed, a workout finishes, or a form problem is detected, the application can send a compact event to a Groq-hosted LLM.

The generated coaching message is then converted into speech using Google Text-to-Speech (gTTS).

The overall pipeline is:

```text
Camera
   ↓
WebRTC
   ↓
MediaPipe Pose Landmarker
   ↓
Pose Landmarks
   ↓
Exercise Detector
   ↓
Rep Counting + Form Analysis
   ↓
Workout Event
   ↓
Groq LLM
   ↓
AI Coaching Text
   ↓
gTTS
   ↓
Voice Feedback
   ↓
Workout History
```

---

# Why This Project

Traditional fitness applications can provide workout plans and exercise instructions, but they usually do not observe how the user is performing the exercise.

This project combines **Computer Vision + Generative AI** to create a more interactive workout experience.

The system separates responsibilities between different components:

### Computer Vision

Observes the user's body through the camera.

### Exercise Detectors

Use pose landmarks, angles, thresholds, and movement states to determine repetitions and form conditions.

### Workout Tracking

Maintains sets, reps, session state, and workout history.

### Generative AI

Converts technical workout events into natural coaching language.

### Text-to-Speech

Turns the generated coaching message into spoken feedback.

An important architectural decision is that the LLM is **not responsible for detecting the user's pose or counting repetitions**.

Pose detection and exercise analysis are handled by MediaPipe and deterministic exercise detectors.

The LLM is used primarily for the **natural-language coaching layer**.

---

# Core Features

## 1. Real-Time Camera Analysis

The application uses `streamlit-webrtc` to receive camera frames directly from the browser.

Each frame is processed through a custom video processor.

The processing pipeline:

1. Receive camera frame.
2. Convert frame to an OpenCV-compatible format.
3. Flip the frame horizontally.
4. Run MediaPipe Pose Landmarker.
5. Extract pose landmarks.
6. Select the currently active exercise detector.
7. Calculate exercise metrics.
8. Draw skeleton and feedback overlays.
9. Store the latest metrics.
10. Return the processed frame to the browser.

---

## 2. Pose Estimation

The project uses the MediaPipe Pose Landmarker model:

```text
ml_models/pose_landmarker_full.task
```

The current configuration uses:

```text
Running Mode:
VIDEO

Minimum Pose Detection Confidence:
0.7

Minimum Pose Presence Confidence:
0.7

Minimum Tracking Confidence:
0.7

Segmentation Masks:
Disabled
```

The pose model provides body landmarks that are then used by the exercise-specific detectors.

---

## 3. Exercise Detection

The application currently supports five exercises:

* Squats
* Push-ups
* Biceps Curls (Dumbbell)
* Shoulder Press
* Lunges

Each exercise has a dedicated detector.

```text
detectors/
├── squat.py
├── pushup.py
├── biceps_curl.py
├── shoulder_press.py
└── lunges.py
```

---

## 4. Rep Counting

Repetitions are calculated using movement states.

For example:

```text
DOWN
  ↓
UP
  ↓
REP + 1
```

Instead of simply checking one frame, the detectors use state transitions.

This reduces the chance of counting the same movement multiple times.

---

## 5. Form Analysis

The system evaluates exercise-specific form conditions.

Examples:

### Squats

* Knee angle
* Back angle
* Depth

### Push-ups

* Elbow angle
* Body alignment
* Hip position

### Biceps Curls

* Elbow angle
* Elbow drift
* Torso swing

### Shoulder Press

* Elbow angle
* Arm extension
* Back arch

### Lunges

* Front knee angle
* Torso angle
* Balance

---

# Supported Exercises

| Exercise       | Rep Signal                        | Form Metrics                            |
| -------------- | --------------------------------- | --------------------------------------- |
| Squats         | Knee angle + movement stage       | Knee angle, back angle, depth           |
| Push-ups       | Elbow angle + movement stage      | Elbow angle, body alignment, hip status |
| Biceps Curls   | Elbow angle + movement stage      | Elbow angle, shoulder stability, swing  |
| Shoulder Press | Elbow angle + movement stage      | Elbow angle, extension, back arch       |
| Lunges         | Front knee angle + movement stage | Knee angle, torso angle, balance        |

---

# How It Works

The application follows this general flow:

```text
┌──────────────────────┐
│    Browser Camera    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Streamlit WebRTC   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ MediaPipe Pose       │
│ Landmarker           │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Pose Landmarks       │
│ + Visibility         │
└──────────┬───────────┘
           │
           ▼
┌────────────────────────────┐
│ Exercise-specific Detector │
└─────────────┬──────────────┘
              │
              ▼
┌────────────────────────────┐
│ Reps + Angles + Form       │
│ Metrics                    │
└─────────────┬──────────────┘
              │
       ┌──────┴───────┐
       │              │
       ▼              ▼
┌─────────────┐  ┌─────────────────┐
│ Streamlit   │  │ Voice Pipeline  │
│ UI          │  └────────┬────────┘
└─────────────┘           │
                          ▼
                 ┌─────────────────┐
                 │ Groq LLM        │
                 │ Llama 3.3 70B   │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ gTTS            │
                 │ Speech          │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Audio Feedback  │
                 └─────────────────┘

Workout Data
     │
     ▼
┌─────────────────┐
│ SQLite Database │
└─────────────────┘
```

---

# System Architecture

The project is divided into multiple logical layers.

## Application Layer

```text
Main App/main.py
```

Responsible for:

* Streamlit configuration
* User login
* Workout planning
* Starting workouts
* Ending workouts
* Camera initialization
* LLM initialization
* Voice pipeline initialization
* Sidebar metrics
* Workout history

---

## Computer Vision Layer

```text
services/vision/exercise_video_processor.py
```

Responsible for:

* Receiving video frames
* MediaPipe initialization
* Pose detection
* Exercise selection
* Detector execution
* Skeleton drawing
* Form overlays
* No-pose warnings
* Latest metrics

---

## Exercise Detection Layer

```text
detectors/
```

Contains:

```text
squat.py
pushup.py
biceps_curl.py
shoulder_press.py
lunges.py
```

Each detector contains exercise-specific movement and form logic.

---

## Core Layer

```text
core/base_exercise.py
```

Provides shared functionality for exercise detectors.

This includes common functionality such as:

* Repetition state
* Landmark point handling
* Angle calculations

---

## AI Coaching Layer

```text
services/coaching/
```

Contains:

```text
llm.py
tts.py
voice_pipeline.py
```

Responsibilities:

### `llm.py`

Communicates with the Groq API.

### `tts.py`

Converts generated text to speech.

### `voice_pipeline.py`

Coordinates:

```text
Workout Event
      ↓
Form Issue Detection
      ↓
LLM
      ↓
Text
      ↓
TTS
      ↓
Audio
```

---

## Persistence Layer

```text
services/persistence/exercise_repository.py
```

Responsible for:

* SQLite connection
* User creation
* User lookup
* Database initialization
* Workout storage
* Workout history retrieval

---

## Configuration Layer

```text
services/config/workout_config.py
```

Contains:

* Exercise options
* Pose connections
* Default metrics
* LLM system prompt

---

## Tracking Layer

```text
services/tracking/metrics.py
```

Responsible for synchronizing the latest vision metrics with Streamlit session state.

---

# Computer Vision Pipeline

## Pose Model

The application loads:

```text
ml_models/pose_landmarker_full.task
```

using MediaPipe Tasks.

The model is configured for video processing.

---

## Frame Processing

Each frame follows:

```text
Camera Frame
     ↓
OpenCV Frame Conversion
     ↓
Horizontal Flip
     ↓
MediaPipe
     ↓
Pose Detection
     ↓
Landmarks
     ↓
Exercise Detector
     ↓
Metrics
     ↓
Visual Overlay
     ↓
Browser
```

---

## Landmark Visibility

The exercise detectors use landmark visibility thresholds.

Most detectors use approximately:

```text
MIN_VISIBILITY = 0.7
```

This prevents the application from relying heavily on joints that are poorly detected or hidden.

---

# Exercise Analysis

## Squats

File:

```text
detectors/squat.py
```

The squat detector calculates:

* Left knee angle
* Right knee angle
* Selected knee angle
* Back angle
* Repetitions
* Depth status

### Rep Logic

```text
Knee angle < 100°
        ↓
      DOWN
        ↓
Knee angle >= 160°
        ↓
      UP
        ↓
    REP + 1
```

### Depth Status

Possible states:

```text
GOOD DEPTH
TOO HIGH
STANDING
N/A
```

The detector also evaluates the user's back angle.

---

# Push-ups

File:

```text
detectors/pushup.py
```

The detector calculates:

* Elbow angle
* Body angle
* Hip deviation
* Repetitions
* Body alignment
* Hip position

### Rep Logic

```text
Elbow angle < 90°
       ↓
     DOWN
       ↓
Elbow angle > 160°
       ↓
      UP
       ↓
   REP + 1
```

### Body Alignment

Possible states:

```text
Straight
Slight Bend
Poor Form
```

### Hip Status

```text
LEVEL
SAGGING
PIKED UP
```

---

# Biceps Curls

File:

```text
detectors/biceps_curl.py
```

The application selects the arm with better elbow visibility.

It calculates:

* Elbow angle
* Elbow drift
* Torso movement
* Repetitions

### Rep Logic

```text
Elbow angle < 50°
      ↓
     UP
      ↓
Elbow angle > 160°
      ↓
    DOWN
      ↓
   REP + 1
```

### Form Analysis

Shoulder / elbow status:

```text
STABLE
ELBOW DRIFTING
```

Swing status:

```text
NO SWING
SWINGING
```

---

# Shoulder Press

File:

```text
detectors/shoulder_press.py
```

The detector calculates:

* Elbow angle
* Arm extension
* Back angle
* Repetitions

### Rep Logic

```text
Elbow angle > 160°
       ↓
      UP
       ↓
Elbow angle < 90°
       ↓
     DOWN
       ↓
    REP + 1
```

### Extension Status

```text
FULL EXTENSION
NEARLY EXTENDED
PRESSING
START POSITION
```

### Back Status

```text
Neutral
Slight Arch
Excessive Arch
```

---

# Lunges

File:

```text
detectors/lunges.py
```

The application calculates both knee angles and uses the smaller angle as the front-knee measurement.

Metrics include:

* Front knee angle
* Torso angle
* Balance
* Repetitions

### Rep Logic

```text
Front knee < 100°
       ↓
      DOWN
       ↓
Front knee > 160°
       ↓
       UP
       ↓
    REP + 1
```

### Balance

```text
BALANCED
OFF BALANCE
```

---

# AI Coaching Pipeline

The AI coaching system is separate from the computer-vision system.

This is an important part of the architecture.

The vision system determines:

```text
"What is happening?"
```

The LLM determines:

```text
"How should I communicate the coaching feedback?"
```

---

## Step 1 — Workout Event

The system can generate events such as:

```text
workout_started
set_completed
workout_completed
no_pose_detected
ongoing_form_check
```

---

## Step 2 — Form Issue Detection

The voice pipeline examines the exercise metrics.

Examples:

### Squat

```text
TOO HIGH
```

or excessive forward leaning.

### Push-up

```text
Poor Form
SAGGING
PIKED UP
```

### Biceps Curl

```text
SWINGING
ELBOW DRIFTING
```

### Shoulder Press

```text
Excessive Arch
Slight Arch
```

### Lunges

```text
OFF BALANCE
```

---

# Step 3 — LLM Prompt

The event is converted into a compact prompt.

For example:

```text
Event: ongoing_form_check
Form Issue: The user's hips are sagging down during the push-up.
```

The LLM receives:

* System prompt
* Recent conversation history
* Current event
* Form issue

---

# Step 4 — LLM Generation

The project uses:

```text
llama-3.3-70b-versatile
```

through Groq.

The system prompt asks the model to generate approximately 10–15 word coaching cues.

The prompt emphasizes:

* Natural spoken language
* Second-person instructions
* High-energy coaching
* Professional tone
* Safety
* Exercise-specific corrections

---

# Step 5 — Text-to-Speech

The generated text is sent to:

```text
gTTS
```

The output is generated as MP3 bytes in memory.

Pipeline:

```text
LLM Text
   ↓
gTTS
   ↓
MP3 Bytes
   ↓
Streamlit Audio
```

---

# Step 6 — Voice Playback

The application uses Streamlit audio playback with autoplay.

The voice pipeline also implements a cooldown for ongoing form corrections.

Current cooldown:

```text
5 seconds
```

This prevents the application from continuously repeating the same correction.

---

# Workout Tracking and Persistence

Workout state is maintained through Streamlit session state.

Tracked information includes:

* Username
* User ID
* Selected exercise
* Target sets
* Reps per set
* Total reps
* Current-set reps
* Completed sets
* Workout start time
* Exercise metrics
* Coaching feedback
* Audio feedback

---

# SQLite Database

The application automatically creates:

```text
Main App/data.db
```

Two tables are created.

## Users

```text
users
├── id
├── username
└── created_at
```

## Exercises

```text
exercises
├── id
├── user_id
├── exercise_name
├── reps
├── sets
├── time
└── created_at
```

---

# Workout History

The application retrieves workout records and uses Pandas to aggregate them by:

```text
Exercise + Date
```

The UI displays:

* Exercise
* Reps
* Sets
* Time
* Date

This provides a simple workout-history dashboard.

---

# User Authentication

The project contains a lightweight login wall.

The user enters a username:

```text
Name (unique)
```

The application then:

1. Searches for the user.
2. Creates the user if they don't exist.
3. Stores the user ID in Streamlit session state.
4. Starts the workout application.

This is intentionally a prototype-level authentication system and should not be considered production-grade authentication.

---

# Project Structure

```text
AI-Gym-Coach/
│
└── ai-gym-coach-main/
    │
    ├── LandingPage/
    │   ├── index.html
    │   ├── style.css
    │   ├── fonts/
    │   ├── IMGs_add_your_own/
    │   └── videos_add_your_own/
    │
    ├── Main App/
    │   │
    │   ├── main.py
    │   ├── requirements.txt
    │   ├── packages.txt
    │   │
    │   ├── core/
    │   │   └── base_exercise.py
    │   │
    │   ├── detectors/
    │   │   ├── squat.py
    │   │   ├── pushup.py
    │   │   ├── biceps_curl.py
    │   │   ├── shoulder_press.py
    │   │   └── lunges.py
    │   │
    │   ├── ml_models/
    │   │   └── pose_landmarker_full.task
    │   │
    │   ├── pages/
    │   │
    │   ├── services/
    │   │   │
    │   │   ├── auth/
    │   │   │   └── login_wall.py
    │   │   │
    │   │   ├── coaching/
    │   │   │   ├── llm.py
    │   │   │   ├── tts.py
    │   │   │   └── voice_pipeline.py
    │   │   │
    │   │   ├── config/
    │   │   │   └── workout_config.py
    │   │   │
    │   │   ├── persistence/
    │   │   │   └── exercise_repository.py
    │   │   │
    │   │   ├── state/
    │   │   │   └── session_defaults.py
    │   │   │
    │   │   ├── tracking/
    │   │   │   └── metrics.py
    │   │   │
    │   │   ├── ui/
    │   │   │   └── style_loader.py
    │   │   │
    │   │   └── vision/
    │   │       └── exercise_video_processor.py
    │   │
    │   ├── static/
    │   │
    │   └── tutorial-info/
    │
    └── README.md
```

---

# Technology Stack

## Frontend / Application

* Streamlit
* streamlit-webrtc
* HTML
* CSS

## Computer Vision

* MediaPipe
* OpenCV
* MediaPipe Pose Landmarker

## Generative AI

* Groq API
* Llama 3.3 70B Versatile
* Prompt Engineering

## Speech

* gTTS

## Data

* SQLite
* Pandas

## Utilities

* python-dotenv
* threading
* Streamlit session state

---

# Requirements

The current `requirements.txt` contains:

```text
streamlit==1.54.0
streamlit-webrtc==0.64.5
mediapipe==0.10.14
opencv-python-headless==4.10.0.84
pandas==2.2.3
groq>=0.12.0
gtts==2.5.3
python-dotenv==1.2.2
```

System packages:

```text
libgl1
libglib2.0-0t64
libsm6
libxext6
```

---

# Installation

## 1. Clone the Repository

```bash
git clone https://github.com/AhmedShaikh45/AI-Gym-Coach.git
cd AI-Gym-Coach
```

---

## 2. Enter the Application Directory

### Windows

```powershell
cd "ai-gym-coach-main\Main App"
```

### macOS / Linux

```bash
cd "ai-gym-coach-main/Main App"
```

---

## 3. Create a Virtual Environment

### Windows

```powershell
python -m venv .venv
.\.venv\Scripts\activate
```

### macOS / Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
```

---

## 4. Install Dependencies

```bash
pip install -r requirements.txt
```

---

# Environment Variables

The generative AI coaching layer requires:

```text
GROQ_API_KEY
```

Create a `.env` file:

```env
GROQ_API_KEY=your_groq_api_key
```

Never commit your real API key to GitHub.

For deployment, use the platform's secret-management system.

The application also checks Streamlit secrets when the environment variable is not available.

---

# Verify the Pose Model

Make sure this file exists:

```text
Main App/ml_models/pose_landmarker_full.task
```

The application loads this model at runtime.

---

# Running the Application

From:

```text
ai-gym-coach-main/Main App
```

run:

```bash
streamlit run main.py
```

Streamlit will provide a local URL.

Open the URL in a browser and allow camera access.

---

# Using the Application

## Step 1 — Login

Enter a username and click:

```text
Start Session
```

---

## Step 2 — Select Exercise

Choose an exercise from the sidebar.

Available options:

```text
Squats
Push-ups
Biceps Curls (Dumbbell)
Shoulder Press
Lunges
```

---

## Step 3 — Set Workout Plan

Choose:

```text
Sets
Reps per Set
```

Then click:

```text
Start Workout
```

---

## Step 4 — Camera

The WebRTC camera interface becomes active.

Position yourself so the required body joints are visible.

---

## Step 5 — Exercise

Perform the selected exercise.

The application calculates:

```text
Pose
  ↓
Angles
  ↓
Movement Stage
  ↓
Reps
  ↓
Form Metrics
```

---

## Step 6 — AI Feedback

If a relevant event or form issue is detected:

```text
Metrics
   ↓
Form Issue
   ↓
Groq LLM
   ↓
Coaching Text
   ↓
gTTS
   ↓
Voice
```

---

## Step 7 — End Workout

Click:

```text
End Workout
```

---

## Step 8 — View History

Your workout history appears below the camera section.

---

# Configuration

Most application configuration is centralized in:

```text
services/config/workout_config.py
```

---

## Exercise List

```python
EXERCISE_OPTIONS = [
    "Squats",
    "Push-ups",
    "Biceps Curls (Dumbbell)",
    "Shoulder Press",
    "Lunges"
]
```

---

# Adding a New Exercise

To add another exercise, the recommended process is:

### 1. Add the exercise

Update:

```text
services/config/workout_config.py
```

### 2. Create a detector

Add:

```text
detectors/new_exercise.py
```

### 3. Implement the detector

The detector should calculate:

* Rep count
* Movement state
* Exercise metrics
* Form status

### 4. Register the detector

Update:

```text
services/vision/exercise_video_processor.py
```

### 5. Add default metrics

Update:

```text
METRICS_FIELDS
```

### 6. Add UI metrics

Update:

```text
main.py
```

### 7. Add coaching rules

Update:

```text
services/coaching/voice_pipeline.py
```

### 8. Add overlays

Update the relevant video overlay logic if needed.

This modular architecture makes adding exercises easier without putting all logic into `main.py`.

---

# Deployment Notes

The project is designed for Streamlit and browser-based WebRTC.

A deployment environment needs:

* Python dependencies
* MediaPipe model
* Browser camera access
* WebRTC support
* System packages
* Groq API key

---

# Browser Camera Permissions

The browser must be allowed to access the camera.

If the camera does not start:

1. Open browser permissions.
2. Allow camera access.
3. Reload the application.
4. Make sure another application is not using the webcam.

---

# WebRTC Networking

The application configures:

```text
stun:stun.l.google.com:19302
```

as a STUN server.

This helps WebRTC establish peer connectivity in many network environments.

---

# Troubleshooting

## Camera Does Not Start

Check:

* Browser camera permissions
* HTTPS requirements for deployment
* `streamlit-webrtc` installation
* Network restrictions
* Camera availability

---

## GROQ API Error

Check:

```text
GROQ_API_KEY
```

Make sure:

* The variable exists.
* The key is valid.
* The application was restarted after setting the key.

---

## Pose Is Not Detected

Try:

* Improving lighting
* Moving farther from the camera
* Keeping the complete body inside the frame
* Reducing body occlusion
* Facing the camera appropriately
* Keeping important joints visible

The application displays:

```text
NO POSE DETECTED
PLEASE FACE THE CAMERA
```

when no pose is detected.

---

## Repetition Count Is Incorrect

Rep counting is threshold-based.

Performance can be affected by:

* Camera position
* Camera angle
* Lighting
* Body orientation
* Occlusion
* Movement speed
* Landmark confidence
* Individual body proportions

Thresholds can be adjusted inside the individual detector files.

---

## Voice Feedback Repeats

Ongoing form feedback currently has a five-second cooldown.

This prevents continuous repetition of the same correction.

Major events such as:

```text
workout_started
set_completed
workout_completed
```

are treated separately.

---

## Workout History Problems

The database is automatically created at:

```text
Main App/data.db
```

Deleting this file resets the locally stored user and workout history.

---

# Limitations

This project is a computer-vision and AI prototype.

It should not be considered a medical system or a replacement for a qualified fitness or healthcare professional.

Current technical limitations include:

* Camera quality affects detection.
* Camera placement affects landmark accuracy.
* Exercise detection uses geometric heuristics and thresholds.
* Different body proportions can affect angle-based analysis.
* Occlusion can reduce detection quality.
* Only one exercise is selected at a time.
* Authentication is username-based and lightweight.
* Workout data is stored locally.
* Voice generation depends on external services.
* LLM responses may vary.
* The system does not provide clinical-grade movement analysis.
* Form feedback should not be treated as medical or injury-prevention advice.

---

# Future Improvements

## Computer Vision

Potential improvements:

* More exercises
* Multi-person detection
* Adaptive thresholds
* Temporal smoothing
* Better repetition-state tracking
* Confidence-aware form scoring
* Camera-view calibration
* Automatic exercise recognition
* Better movement segmentation

---

## Generative AI

Potential improvements:

* Personalized coaching memory
* Personalized workout plans
* Session summaries
* Adaptive coaching
* Multi-language coaching
* More detailed workout explanations
* Lower-latency LLM feedback
* Personalized coaching styles

---

## Analytics

Potential improvements:

* Weekly dashboards
* Monthly progress
* Personal records
* Exercise consistency
* Form-quality trends
* Workout intensity
* Progress charts
* Workout reports

---

## Product

Potential improvements:

* Secure authentication
* User profiles
* Cloud database
* Mobile interface
* Cloud workout history
* Exportable reports
* Custom voice selection
* Multiple AI coaching personalities
* Better audio controls

---

# Contributing

Contributions are welcome.

Create a feature branch:

```bash
git checkout -b feature/your-feature
```

Make your changes and commit:

```bash
git add .
git commit -m "Add your feature"
```

Push:

```bash
git push origin feature/your-feature
```

Then open a pull request.

For new exercise detectors, keep exercise-specific logic inside:

```text
detectors/
```

rather than placing large blocks of exercise logic inside `main.py`.

---

# License

No explicit license file is currently included in this repository.

Until a license is added, do not assume that the project is released under an open-source license.

---

# Author

## Ahmed Shaikh

GitHub:

https://github.com/AhmedShaikh45

Project:

https://github.com/AhmedShaikh45/AI-Gym-Coach

---

# Project Summary

This project demonstrates how several AI engineering technologies can be combined into one end-to-end application:

```text
Computer Vision
       +
Pose Estimation
       +
Exercise Detection
       +
Geometric Analysis
       +
State Tracking
       +
Generative AI
       +
Prompt Engineering
       +
Text-to-Speech
       +
Database Persistence
       =
Real-Time AI Fitness Coach
```

The project demonstrates practical experience with:

* Computer Vision
* MediaPipe
* OpenCV
* Real-time video processing
* Pose estimation
* Exercise recognition
* Rule-based movement analysis
* Generative AI
* LLM integration
* Prompt engineering
* Groq API
* Llama models
* Text-to-Speech
* Streamlit
* WebRTC
* SQLite
* Pandas
* Python application architecture

It is designed as an end-to-end AI application rather than a standalone machine-learning notebook.
