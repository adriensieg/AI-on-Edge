# AI-on-Edge
How to run AI on Edge


## Infrastructure Set Up

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
- Choose teacher (original YOLO model) and student (MobileNet+YOLO-like) architecture.
- Use knowledge distillation loss: soft logits + ground truth.
- Train student model with distilled dataset.
- Evaluate F1/accuracy tradeoff.

### Quantization
- Apply Post-Training Quantization (PTQ) using torch.quantization or ONNX Runtime.
- Test INT8 and FP16 variants.
- Benchmark inference latency and accuracy.
- Validate quantized model with edge test set.

## Model Format and Export
- Choose final model format: ONNX
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















