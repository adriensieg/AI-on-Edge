

### The key rule

For every image - `images/train/frame_000123.jpg` we must have **a matching label file** `labels/train/frame_000123.txt`.
If there is **no object in the image**, the `.txt` file **should usually exist but be empty**.

### YOLO label file format

Each `.txt` file contains one line per object:

```
dataset/
├── images/
│   ├── train/
│   │   ├── frame_000001.jpg
│   │   ├── frame_000002.jpg
│   │   └── ...
│   ├── val/
│   │   ├── frame_000101.jpg
│   │   └── ...
│   └── test/               # optional
│       ├── frame_000201.jpg
│       └── ...
├── labels/
│   ├── train/
│   │   ├── frame_000001.txt
│   │   ├── frame_000002.txt
│   │   └── ...
│   ├── val/
│   │   ├── frame_000101.txt
│   │   └── ...
│   └── test/               # optional
│       ├── frame_000201.txt
│       └── ...
└── data.yaml
```
