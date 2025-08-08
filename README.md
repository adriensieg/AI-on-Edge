# AI-on-Edge
How to run AI on Edge

## Infrastructure Set Up
- IP cameras (AXIS P3215 and AXIS P3265-LV) capture real-time video.
- Each camera provides an **RTSP stream** accessible via a **local network**.
- The cameras are connected via **PoE Ethernet** to avoid power limitations.
- **RTSP Connector (OpenCV)**: Opens RTSP streams and continuously reads frames.
- **Frame Buffer** (**Multi-Threaded Queue**): Stores frames **in memory** before processing.
- **Frame Preprocessing**: Frames are resized (e.g., **640x360**), converted to a suitable format, and optionally timestamped.
- **Publish to Processing Queue**: Frames are pushed to a **queue** for backend processing.
- **Frame Fetcher** (Async Queue): Retrieves frames from the processing queue.
- Raw frames are stored in H.264/H.265 format for efficient storage.
- Rolling Buffer Management: Deletes old footage when storage reaches capacity

- RTSP camera sends a continuous video stream
  - Resolution: 1080p (1920×1080)
  - FPS: 30 (standard CCTV)
  - Encoding: H.264
  - Bitrate: ~4 Mbps per stream

## Data Collection & Labeling

- Extract frames from **RTSP footage** at **1 fps** using `ffmpeg` or `OpenCV`
- Store **raw extracted frames** in structured directories (`/raw/footage/YYYY-MM-DD`)
- Define **labeling guidelines** (e.g., bounding box includes full fry, occluded fries allowed, min size threshold).
- Use a label tool with `YOLOv8`/ `YOLOv5` annotation format
- Coordinates are normalized (0 to 1) relative to image `width/height`.
- Create **versioned dataset management** (`dataset_v1`, `dataset_v2`)
- **Split dataset**: `train/val/test` using stratified sampling
- Store dataset in **GCS bucket**

## Model Training with YOLO

- Choose YOLO variant: `YOLOv8n` or YOLOv5s (lightweight for edge deployment)
- Configure training hyperparameters (`batch_size`, `img_size`, `epochs`, `lr, optimizer`)
- Configure `data.yaml` with number of classes, class names, paths → `Small`, `Medium`, `Large`
- Use `ultralytics` or `YOLOv5` CLI/SDK for training → **YoloV5 is open source**
- Use GCP Vertex AI or **Kubeflow Pipelines** for orchestration
- Pipeline steps: `preprocess` → `train` → `eval` → `export` → `compress` → `upload`
- Save final weights (`best.pt`, `last.pt`).
- Export frozen inference model to `ONNXformat`
- Log experiment metadata using MLflow

## Model Optimization

### Model Distillation
- Choose teacher (original `YOLO` model) and student (`MobileNet+YOLO`-like) architecture.
- Use knowledge distillation `loss`: `soft logits` + `ground truth`.
- Train student model with distilled dataset.
- Evaluate F1/accuracy tradeoff.

### Quantization
- Apply Post-Training Quantization (PTQ) using torch.quantization or ONNX Runtime.
- Test `INT8` and `FP16` variants.
- Benchmark inference latency and accuracy.
- Validate quantized model with edge test set.

## Model Format and Export
- Choose final model format: `ONNX`
- Store final model artifacts in GCS with version tags

## Inference
- Use `FastAPI` as inference API server.
- Configure camera reader:
  - Use `cv2.VideoCapture(RTSP_URL)`
  - Use decoding on separate thread to avoid blocking
- Inference pipeline:
  - RTSP stream → frame capture (OpenCV)
  - Preprocess (resize, normalize)
  - YOLO inference (ONNX Runtime / TorchScript CPU)
  - Postprocess (NMS, fry counting)
  - Store prev_count state
  - Publish to NATS if abs(count - prev_count) ≥ 1
- Use multiprocessing.Process for RTSP capture and model inference
- Use threading.Thread for NATS publisher and health checks.
- Apply optimizations:
  - RTSP read buffer management
  - Frame skipping / sampling
  - Batch inference (if multiple cams)
  - Use `multiprocessing.Queue` for decoupled pipeline stages
- Optimize pipeline with `queue.Queue` for inter-process communication.


## Event-Driven NATS Publisher
- Install and configure `nats-py` client.
- Define topic: fries.station.count.update
- Payload schema: {"timestamp": "...", "count": n, "delta": ±1}
- Only publish on count change event.
- Add retries and logging on publish failures
- Secure with authentication if needed

## Web Visualization Service
- Web app using FastAPI + Jinja2 or ReactJS frontend
- Show current fries count, delta, last change timestamp
- Provide optional live video stream with overlay (bbox visualization)
- `RTSP` to `MJPEG` proxy via `ffmpeg` or `opencv`→`FastAPI` stream.
- REST endpoints:
  - `/count` – return latest count
  - `/stream` – return MJPEG stream
  - `/events` – list recent count change events
 
## Containerization and Deployment
- Create Dockerfile for app:
  - Use python:3.11-slim
  - Install `fastapi`, `uvicorn`, `opencv`, `nats-py`, `onnxruntime`, etc.
- Build Docker image and push to GCR.
- Define Kubernetes manifests:
  - `Deployment`: edge app
  - `Service`: expose internal API
  - `ConfigMap`: model path, NATS URL
  - `PVC`: persistent volume for logs
- Use Helm or kustomize for versioned deployment.
- Apply Kubernetes resource limits:
  - `requests`: `{cpu: 500m, memory: 512Mi}`
  - `limits`: `{cpu: 2, memory: 2Gi}`

## Monitoring and Logging
- Add structured logs with loguru or structlog
- Log each inference, RTSP failure, publish event
- Integrate with GCP Logging
- Add New Relic metrics:
  - inference_duration_seconds
  - frames_processed_total
  - count_change_events_total
Expose /metrics endpoint for scraping.

## Performance Estimation & Optimization
- Benchmark model size, latency, and memory footprint (CPU-only)
- Run edge inference benchmark: FPS, CPU usage, latency
- Estimate hardware requirements:
  - Image size: 640x640
  - Inference time: <300ms/frame
  - CPU: 4-core minimum
  - RAM: 4 GB minimum
  - Bandwidth: negligible (local-only inference)

## CI/CD and Logging
- Implement CI pipeline (GitHub Actions or Cloud Build)
- Unit test: inference, RTSP stream, event publishing
- Logging:

Log model predictions with timestamps

Log NATS events sent

 

 















