# YOLO-like model deployment on K3s on Raspberry Pi 4

## Objectives
- The primary **business objective** is to assist users in **accurately identifying the appropriate waste bin for disposal when uncertain**.
- The system will be **deployed on a Raspberry Pi** equipped with a **5MP camera**.
- Users will **present the waste item** to the **camera**, and the **Raspberry Pi** will **infer the correct bin category** using the trained model.
- The **YOLOv8-nano trash detector** against an **RTSP camera** will run on a **Raspberry Pi 4 (8 GB)** using **k3s (lightweight Kubernetes)** with one node. 

**Conditions**: 
- I trained a **YOLOv8-nano model** using a **Garbage Classification Dataset**
- The dataset comprises **2,467 labeled images** across six categories:
  - Cardboard (393 images)
  - Glass (491 images)
  - Metal (400 images)
  - Paper (584 images)
  - Plastic (472 images)
  - Trash (127 images)
 
## Deployment

- I want to deploy the model on a Kubernetes cluster K3s for Raspberry pi 4 with one node
  - https://medium.com/@stevenhoang/step-by-step-guide-installing-k3s-on-a-raspberry-pi-4-cluster-8c12243800b9
- I want to deploy my application using **GGitOps with ArgoCD**G

 ## Features:
 - Pulls our YOLOv8-nano model (ONNX for speed and simplicity on Pi CPU),
 - Grabs frames from our RTSP camera,
 - Does real-time inference on CPU,
 - Exposes an HTTP MJPEG stream of annotated frames (and a JSON endpoint) via Traefik Ingress (bundled with k3s) or NodePort.
 - The end user can view annotated video frames on a display screen.
 - A web application built with Python (FastAPI) and HTML/vanilla JavaScript is exposed, enabling real-time visualization of the annotated frames, including the detected garbage category label. - Counts how many objects have been classified per category

## Main concepts:
- OS prep
- k3s install
- Container build (arm64)
- FastAPI app (RTSP → ONNX → annotate → MJPEG/JSON)
- Kubernetes manifests:
  - Deployment
  - Service
  - Ingress
  - Config
  - Secret
  - HPA
  - PodSecurity
  - NetworkPolicy)
- GitOps repo layout + Argo CD Application
- Monitoring/logging
- Perf tuning, and troubleshooting.

 - 64-bit OS, cgroups enabled, governor performance
 - k3s installed (Traefik enabled)
 - App builds for linux/arm64, runs locally
 - Push image to registry
 - Argo CD installed; Application synced to repo
 - RTSP reachable; MJPEG live at `/mjpeg`; JSON at `/detections`
 - Business logic: class → bin label shown in UI
 - Metrics scraped and logs healthy
 - Thermals OK under sustained load

