# Computer Vision on Edge

## The layout of the project

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
