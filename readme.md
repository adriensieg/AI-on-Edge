# Computer Vision on Edge


# The layout of the project

## Core Infrastructure
- `main.py`: Clean entry point that simply creates and runs the app
- `app/core/config.py`: Centralized configuration using Pydantic Settings
- `app/core/logging_config.py`: Structured logging setup

## Data Models
- `app/models/detection.py`: Pydantic models for type-safe API responses and data validation

## API Layers
- `app/api/main.py`: FastAPI application factory with lifespan management
- Route modules: Separated by functionality (web interface, video streaming, REST API)

## Service

- `VideoProcessor`: Handles YOLO model loading and frame processing
- `CameraManager`: Manages camera initialization, configuration, and frame capture
- `StreamManager`: Orchestrates the entire video streaming pipeline
- `WebSocketManager`: Handles real-time WebSocket connections

```
yolo-detection-system/
├── main.py                         # Application entry point
├── requirements.txt                # Project dependencies
├── .env                            # Environment variables (create manually)
├── README.md                       # Project documentation
│
├── app/
│   ├── __init__.py
│   │
│   ├── core/                       # Core configuration and utilities
│   │   ├── __init__.py
│   │   ├── config.py              # Configuration settings
│   │   └── logging_config.py      # Logging configuration
│   │
│   ├── models/                     # Data models and schemas
│   │   ├── __init__.py
│   │   └── detection.py           # Pydantic models for API responses
│   │
│   ├── services/                   # Business logic and services
│   │   ├── __init__.py
│   │   ├── video_processor.py     # YOLO video processing
│   │   ├── camera_manager.py      # Camera initialization and management
│   │   ├── stream_manager.py      # Video streaming orchestration
│   │   ├── performance_monitor.py # Monitor performance
│   │   └── websocket_manager.py   # WebSocket connection management
│   │
│   └── api/                       # API routes and endpoints
│       ├── __init__.py
│       ├── main.py                # FastAPI app factory
│       └── routes/                # Route modules
│           ├── __init__.py
│           ├── web.py             # Web interface routes
│           ├── video.py           # Video streaming routes
│           └── api.py             # REST API routes
│
├── static/                        # Static files (CSS, JS, images)
├── templates/                     # Jinja2 HTML templates
├── models/                        # YOLO model files
└── logs/                          # Application logs
```
