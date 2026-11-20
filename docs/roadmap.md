# Development Roadmap

Computer Vision-Based Customer Traffic Monitoring: from a MOT17 research project to a full-stack customer traffic monitoring application.

Work is tracked on GitHub with milestones, issues, labels and the **Pedestrian Tracking Board** project. This document is the written plan; the issues are the source of truth for task status.

## 1. Overview and objectives

The project analyzes customer movement in video footage. The research phase used annotated MOT17 data (ground-truth bounding boxes and pedestrian IDs) to engineer movement features, train trajectory prediction models, count pedestrians and build traffic heatmaps.

The goal of this roadmap is to remove the dependency on manual annotations and turn the research into a usable application:

1. Automatically detect (YOLO) and track (ByteTrack) pedestrians in raw video.
2. Feed those tracks into the existing feature extraction, trained models and analytics.
3. Expose everything through a React + FastAPI web application.
4. Persist results (PostgreSQL + object storage), test, benchmark, evaluate and deploy.
5. Optionally, support live cameras.

## 2. Current state of the repository

Verified on 2026-11-19 by inspecting the repository:

| Area | Status |
|---|---|
| MOT17 ground truth and detections (`data/processed/MOT17-GT-Files/`) | Present: MOT17-04, -09, -10, -11, -13 |
| Preprocessed feature CSVs (`MOT17_04_pedestrians_preprocessed.csv`, `MOT17_pedestrians_merged.csv`) | Present: center/foot points, box area, aspect ratio, normalized position, time, per-frame count, displacement, speed, and standard-scaled columns |
| Research paper (`docs/research_paper.pdf`) | Present; kept unchanged as the ML foundation |
| Preprocessing, modeling and evaluation notebooks | **Not yet in the repo.** `notebooks/*.ipynb` are empty placeholders |
| Trained KNN, SVM/SVR, Random Forest, Gradient Boosting models and scalers | **Not yet in the repo.** `models/` folders are empty |
| Counting and heatmap code | **Not yet in the repo** |
| `backend/`, `scripts/`, `tests/`, `docs/*.md`, `Dockerfile`, CI workflow | Empty scaffolding only |
| `frontend/` | Empty placeholder |

The ML work exists outside the repository. Importing it is the first issue of Milestone 2 and a prerequisite for all model integration work.

## 3. Planned features

- Automatic pedestrian detection (YOLO) and multi-object tracking (ByteTrack) from MP4 input.
- Annotated output video with boxes, track IDs and predicted positions.
- Tracking CSV in MOT17-compatible format.
- Automatic feature extraction and next-position prediction with the existing models.
- Pedestrian counts (per-frame occupancy, unique tracks, time buckets) and foot-position heatmaps.
- Web app: upload, background processing with status, results dashboard, CSV export.
- PostgreSQL metadata + object storage for video, retention policy, CI, Docker, deployed demo.
- Optional: webcam/RTSP input, live dashboard, line crossing, zones and dwell time, sequence models.

## 4. Proposed architecture

```
                 ┌──────────────── React frontend (Vite) ────────────────┐
                 │ Upload · Jobs/status · Results (video, Plotly charts) │
                 └───────────────────────────┬───────────────────────────┘
                                             │ REST (/api/v1), later WebSockets
                 ┌───────────────────────────▼───────────────────────────┐
                 │                    FastAPI backend                    │
                 │ upload & validation · job queue · results · exports   │
                 └───────┬──────────────────────┬─────────────────┬──────┘
                         │ background job       │ metadata        │ files
         ┌───────────────▼──────────────┐ ┌─────▼──────┐  ┌───────▼────────┐
         │       Analysis pipeline      │ │ PostgreSQL │  │ Object storage │
         │ OpenCV read → YOLO detect →  │ │ videos,    │  │ raw video,     │
         │ ByteTrack → track cleaning → │ │ jobs,      │  │ annotated MP4, │
         │ features → model predict →   │ │ analytics, │  │ CSVs, heatmaps │
         │ counts + heatmap → annotate  │ │ object keys│  │                │
         └──────────────────────────────┘ └────────────┘  └────────────────┘
```

Rules:

- Raw video is never stored in PostgreSQL; the database only holds metadata, analytics and object keys.
- The web backend calls the same `analyze_video()` pipeline function as the CLI (`scripts/run_pipeline.py`).
- Inference uses the saved training scaler and documented feature order; it never refits.
- Displacement and speed are computed per track ID only; gaps and invalid boxes are handled before prediction.
- Predictions are labeled with the horizon the models were trained for. Long-horizon trajectories are not claimed unless implemented and evaluated.

## 5. Milestones and timeline

| Milestone | Objective | Estimate | Completion criteria |
|---|---|---|---|
| **M1** Automatic Video Detection and Tracking | Raw MP4 → boxes, track IDs, annotated video, tracking CSV | 1–2 weeks | An MP4 produces an annotated video and a tracking CSV |
| **M2** Machine Learning Pipeline Integration | Tracks → features → predictions, counts, heatmaps | 1–2 weeks | A raw video runs through the full pipeline with no ground-truth annotations |
| **M3** Full-Stack Web Application | Upload, process and view results in the browser | 2–3 weeks | A user uploads footage and sees detections, predictions, counts and heatmaps |
| **M4** Database, Cloud Storage, Testing, Deployment | Persistence, CI, Docker, benchmarks, evaluation, demo | 1–2 weeks | Results persist, past jobs load, tests pass in CI, a documented demo is deployed |
| **M5** Advanced Real-Time Monitoring (optional) | Live camera input and live analytics | Open-ended | A live source updates analytics continuously |

Estimated total for the initial usable application (M1–M4): **5–9 weeks**.

## 6. Technology stack

| Layer | Technology |
|---|---|
| Video I/O | OpenCV |
| Detection | Ultralytics YOLO (YOLOv8n / YOLO11n to start) |
| Tracking | ByteTrack |
| ML | scikit-learn (KNN, SVR, Random Forest, Gradient Boosting), pandas, NumPy, joblib |
| Backend | FastAPI, Uvicorn, SQLAlchemy, Alembic |
| Frontend | React (Vite), Plotly |
| Data | PostgreSQL, S3-compatible object storage (Amazon S3 or Supabase Storage) |
| Ops | Docker, docker compose, GitHub Actions, pytest |

## 7. Dependencies between phases

- **M1 → M2:** model integration needs the tracking CSV format from M1.
- **Import of existing ML work (first M2 issue)** blocks feature refactoring, model loading, counting and heatmaps.
- **M2 → M3:** the backend job runner calls the M2 end-to-end pipeline. Frontend and FastAPI scaffolding can start in parallel with M1/M2.
- **M3 → M4:** persistence replaces M3's in-memory job storage. Docker and CI can start as soon as the backend and tests exist.
- **M5** depends only on M1 video input and M3/M4 infrastructure; nothing in M1–M4 depends on M5.

## 8. Initial usable application

The initial application is done when M1–M4 high-priority issues are closed and:

- A user can open the deployed web app, upload an MP4, and watch its status move to finished.
- The results page shows the annotated video with track IDs and predicted positions, occupancy over time, summary counts and a heatmap.
- Tracking, prediction and count CSVs can be downloaded.
- Past jobs can be reopened after a restart.
- CI passes and `docs/results.md` reports benchmark and ADE/FDE results against constant-position and constant-velocity baselines.

## 9. Optional future improvements

Milestone 5 (webcam, RTSP, WebSocket live view, live counts, line crossing, zones and dwell time, LSTM/sequence models, streaming benchmarks), plus ideas not yet planned as issues: multi-camera support, re-identification across cameras, perspective/floor-plan mapping of heatmaps, and per-store dashboards.

## 10. Issues by milestone

Board: [Pedestrian Tracking Board](https://github.com/users/Card1n/projects/2) · [All milestones](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/milestones)

### M1 — Automatic Video Detection and Tracking

| Issue | Title | Priority | Depends on |
|---|---|---|---|
| [#1](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/1) | [Video] Implement video input and frame extraction with OpenCV | high | — |
| [#2](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/2) | [Detection] Implement YOLO pedestrian detection | high | [#1](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/1) |
| [#3](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/3) | [Tracking] Integrate ByteTrack for persistent pedestrian IDs | high | [#2](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/2) |
| [#4](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/4) | [Video] Export annotated video with bounding boxes and track IDs | high | [#3](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/3) |
| [#5](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/5) | [Tracking] Export tracking data as MOT17-compatible CSV | high | [#3](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/3) |
| [#6](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/6) | [Testing] Tests for video input, detection output and tracking data | high | [#1](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/1), [#2](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/2), [#5](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/5) |

### M2 — Machine Learning Pipeline Integration

| Issue | Title | Priority | Depends on |
|---|---|---|---|
| [#7](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/7) | [ML] Import existing notebooks, trained models and scaler into the repository | high | — |
| [#8](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/8) | [Preprocessing] Refactor feature extraction from notebooks into a Python module | high | [#7](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/7) |
| [#9](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/9) | [Preprocessing] Convert YOLO/ByteTrack output into the model feature format | high | [#5](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/5), [#8](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/8) |
| [#10](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/10) | [ML] Load trained KNN, SVR, Random Forest and Gradient Boosting models | high | [#7](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/7) |
| [#11](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/11) | [ML] Implement pedestrian movement inference interface | high | [#9](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/9), [#10](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/10) |
| [#12](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/12) | [Analytics] Visualize predicted vs observed pedestrian positions | medium | [#4](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/4), [#11](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/11) |
| [#13](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/13) | [Analytics] Integrate pedestrian counting into the pipeline | high | [#7](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/7), [#5](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/5) |
| [#14](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/14) | [Analytics] Integrate heatmap generation into the video pipeline | high | [#7](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/7), [#5](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/5) |
| [#15](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/15) | [Pipeline] End-to-end video analysis pipeline entry point | high | [#11](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/11), [#12](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/12), [#13](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/13), [#14](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/14) |
| [#16](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/16) | [Testing] Tests for feature extraction and model inference | high | [#8](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/8), [#9](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/9), [#11](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/11) |

### M3 — Full-Stack Web Application

| Issue | Title | Priority | Depends on |
|---|---|---|---|
| [#17](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/17) | [Frontend] Initialize React frontend project structure | high | — |
| [#18](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/18) | [Backend] Initialize FastAPI application and core API structure | high | — |
| [#19](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/19) | [Upload] Implement video upload (API endpoint and UI) | high | [#17](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/17), [#18](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/18) |
| [#20](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/20) | [Backend] Process uploaded videos as background jobs | high | [#15](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/15), [#19](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/19) |
| [#21](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/21) | [Backend] Processing status, progress and error handling for uploads | high | [#20](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/20) |
| [#22](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/22) | [Frontend] Dashboard for viewing annotated videos | high | [#20](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/20) |
| [#23](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/23) | [Frontend] Display predicted pedestrian movement and track IDs | medium | [#22](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/22), [#12](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/12) |
| [#24](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/24) | [Frontend] Display pedestrian counts and traffic heatmaps | high | [#22](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/22), [#13](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/13), [#14](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/14) |
| [#25](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/25) | [Backend] CSV export for tracking and analytics results | medium | [#20](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/20) |
| [#26](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/26) | [Testing] Integration tests for frontend/backend communication | medium | [#19](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/19), [#20](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/20), [#21](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/21) |

### M4 — Database, Cloud Storage, Testing, and Deployment

| Issue | Title | Priority | Depends on |
|---|---|---|---|
| [#27](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/27) | [Database] Design PostgreSQL schema for videos, jobs, tracks and analytics | high | [#21](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/21) |
| [#28](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/28) | [Database] Implement database models and migrations | high | [#27](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/27) |
| [#29](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/29) | [Storage] Store raw and processed videos in object storage | high | [#19](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/19) |
| [#30](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/30) | [Database] Link stored video objects to database records | high | [#28](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/28), [#29](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/29) |
| [#31](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/31) | [Backend] API endpoints to retrieve previously processed analyses | medium | [#30](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/30) |
| [#32](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/32) | [Backend] File validation, access control and footage retention policy | medium | [#30](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/30) |
| [#33](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/33) | [Deployment] Containerize the backend with Docker | high | [#18](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/18) |
| [#34](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/34) | [Deployment] Run automated tests with GitHub Actions | high | [#6](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/6) |
| [#35](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/35) | [ML] Benchmark inference performance (FPS and latency) | medium | [#15](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/15) |
| [#36](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/36) | [ML] Evaluate trajectory prediction with ADE/FDE against baselines | high | [#11](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/11) |
| [#37](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/37) | [Deployment] Deploy a functional demo | high | [#31](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/31), [#33](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/33), [#34](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/34) |
| [#38](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/38) | [Docs] Update README, installation instructions and technical documentation | medium | [#37](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/37) |

### M5 — Advanced Real-Time Monitoring (Optional)

| Issue | Title | Priority | Depends on |
|---|---|---|---|
| [#39](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/39) | [Video] Add webcam input support | low | [#1](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/1) |
| [#40](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/40) | [Video] Implement RTSP camera stream ingestion | low | [#39](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/39) |
| [#41](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/41) | [Backend] Live processing and dashboard updates over WebSockets | low | [#40](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/40), [#18](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/18), [#17](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/17) |
| [#42](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/42) | [Frontend] Continuously updated counts and movement predictions | low | [#41](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/41) |
| [#43](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/43) | [Analytics] Configurable line-crossing detection | low | [#5](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/5) |
| [#44](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/44) | [Analytics] Zone-based traffic analytics and approximate dwell time | low | [#5](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/5) |
| [#45](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/45) | [ML] Explore LSTM or other sequence models for trajectory prediction | low | [#36](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/36) |
| [#46](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/46) | [Deployment] Benchmark end-to-end streaming latency and throughput | low | [#41](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/41) |

## 11. Recommended implementation order

Follows the issue dependencies. Items on the same line can be worked on in parallel.

1. **[#1](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/1)** video input → **[#2](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/2)** YOLO detection → **[#3](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/3)** ByteTrack → **[#5](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/5)** tracking CSV and **[#4](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/4)** annotated video → **[#6](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/6)** M1 tests.
2. In parallel with M1, bring the existing ML work into the repo: **[#7](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/7)**. It needs nothing from M1 and blocks most of M2.
3. **[#8](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/8)** feature module and **[#10](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/10)** model loading → **[#9](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/9)** tracks-to-features → **[#11](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/11)** inference.
4. **[#13](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/13)** counting and **[#14](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/14)** heatmap (only need [#5](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/5) and [#7](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/7)) → **[#12](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/12)** prediction overlay → **[#15](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/15)** end-to-end pipeline → **[#16](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/16)** M2 tests.
5. Start **[#34](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/34)** CI as soon as [#6](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/6) lands so every later change is tested.
6. **[#17](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/17)** React and **[#18](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/18)** FastAPI setup (can start any time) → **[#19](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/19)** upload → **[#20](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/20)** background jobs (needs [#15](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/15)) → **[#21](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/21)** status → **[#22](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/22)** video dashboard → **[#24](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/24)** analytics, **[#23](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/23)** predictions, **[#25](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/25)** CSV export → **[#26](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/26)** integration tests.
7. **[#27](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/27)** schema → **[#28](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/28)** models/migrations, **[#29](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/29)** object storage → **[#30](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/30)** linking → **[#31](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/31)** history API and **[#32](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/32)** validation and retention.
8. **[#33](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/33)** Docker, **[#35](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/35)** benchmarks and **[#36](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/36)** ADE/FDE evaluation → **[#37](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/37)** deploy → **[#38](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/38)** docs.
9. M5 ([#39](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/39)–[#46](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/46)) only after the initial application is done.

**First issue to work on: [#1](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/1), video input and frame extraction with OpenCV.** It has no dependencies and every other stage of the pipeline builds on it. Do [#7](https://github.com/Card1n/Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project/issues/7) alongside it, because only you have the notebooks and models.
