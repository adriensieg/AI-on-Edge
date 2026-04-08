
# 0. Details
### The key rule

For every image - `images/train/frame_000123.jpg` we must have **a matching label file** `labels/train/frame_000123.txt`.
If there is **no object in the image**, the `.txt` file **should usually exist but be empty**.

### Layout
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

### YOLO label file format

Each .txt file contains one line per object:

class_id x_center y_center width height

Example:

0 0.512 0.473 0.182 0.291
1 0.234 0.620 0.090 0.140

Meaning:

- 0 = first class
- 1 = second class
- coordinates are normalized between 0 and 1
- Format is:
  - center x
  - center y
  - box width
  - box height
 

# 1. Implementation

### Create a PIP environment
```
python3 venv -m venv-uhc
```

```
mkdir -p dataset/images/train dataset/images/val dataset/images/test
mkdir -p dataset/labels/train dataset/labels/val dataset/labels/test
```

python extract_frames.py


























