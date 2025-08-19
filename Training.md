# YOLOv8 Mathematical Transformations

<img width="978" height="1024" alt="image" src="https://github.com/user-attachments/assets/eb935038-94be-4cf1-9c4b-717447f6772a" />

## Input Image
- **Input Size**: 640×640×3
- **Explanation**: RGB image with width=640, height=640, channels=3

## Backbone Network Transformations

### Step 0: First Convolution
- **Input**: 640×640×3
- **Operation**: Conv k=3, s=2, p=1
- **Output**: 320×320×64

**Math**: 
- Formula: `output_size = (input_size + 2×padding - kernel_size) / stride + 1`
- Width/Height: `(640 + 2×1 - 3) / 2 + 1 = (640 + 2 - 3) / 2 + 1 = 639/2 + 1 = 319.5 + 1 = 320.5 → 320`
- Channels: Determined by number of filters = 64

### Step 1: Second Convolution  
- **Input**: 320×320×64
- **Operation**: Conv k=3, s=2, p=1
- **Output**: 160×160×128

**Math**:
- Width/Height: `(320 + 2×1 - 3) / 2 + 1 = 319/2 + 1 = 160.5 → 160`
- Channels: 128 (set by layer configuration)

### Step 2: C2f Block
- **Input**: 160×160×128
- **Operation**: C2f with shortcut=True, n=3, c=d
- **Output**: 160×160×128

**Math**:
- C2f blocks maintain spatial dimensions when stride=1
- Width/Height: 160×160 (unchanged)
- Channels: 128 (unchanged, as c=d means output channels = input channels)

### Step 3: Convolution with Stride 2
- **Input**: 160×160×128
- **Operation**: Conv k=3, s=2, p=1  
- **Output**: 80×80×256

**Math**:
- Width/Height: `(160 + 2×1 - 3) / 2 + 1 = 159/2 + 1 = 80.5 → 80`
- Channels: 256

### Step 4: C2f Block
- **Input**: 80×80×256
- **Operation**: C2f shortcut=True, n=6, c=d
- **Output**: 80×80×256

**Math**:
- Spatial dimensions unchanged (stride=1): 80×80
- Channels unchanged: 256

### Step 5: Convolution with Stride 2
- **Input**: 80×80×256
- **Operation**: Conv k=3, s=2, p=1
- **Output**: 40×40×512

**Math**:
- Width/Height: `(80 + 2×1 - 3) / 2 + 1 = 79/2 + 1 = 40.5 → 40`
- Channels: 512

### Step 6: C2f Block
- **Input**: 40×40×512
- **Operation**: C2f shortcut=True, n=6, c=d
- **Output**: 40×40×512

**Math**:
- Spatial dimensions unchanged: 40×40
- Channels unchanged: 512

### Step 7: Convolution with Stride 2
- **Input**: 40×40×512
- **Operation**: Conv k=3, s=2, p=1
- **Output**: 20×20×512

**Math**:
- Width/Height: `(40 + 2×1 - 3) / 2 + 1 = 39/2 + 1 = 20.5 → 20`
- Channels: 512

### Step 8: C2f Block
- **Input**: 20×20×512
- **Operation**: C2f shortcut=True, n=3, c=d
- **Output**: 20×20×512

**Math**:
- Spatial dimensions unchanged: 20×20
- Channels unchanged: 512

### Step 9: SPPF (Spatial Pyramid Pooling Fast)
- **Input**: 20×20×512
- **Operation**: SPPF
- **Output**: 20×20×512

**Math**:
- SPPF maintains spatial dimensions
- Multiple pooling operations are concatenated but result in same channel count
- Output: 20×20×512

## Head Network Transformations

### Upsampling Operations

#### Step 10: Upsample
- **Input**: 20×20×512
- **Operation**: Upsample (scale_factor=2)
- **Output**: 40×40×512

**Math**:
- Width/Height: `20 × 2 = 40`
- Channels unchanged: 512

#### Step 11: Concat
- **Input1**: 40×40×512 (from upsample)
- **Input2**: 40×40×512 (from step 6, P4)
- **Operation**: Concatenation along channel dimension
- **Output**: 40×40×1024

**Math**:
- Width/Height: 40×40 (unchanged)
- Channels: `512 + 512 = 1024`

#### Step 12: C2f Block
- **Input**: 40×40×1024
- **Operation**: C2f shortcut=False, n=3, c=d
- **Output**: 40×40×512

**Math**:
- Spatial dimensions unchanged: 40×40
- Channels reduced to 512 (feature compression)

#### Step 13: Upsample
- **Input**: 40×40×512
- **Operation**: Upsample (scale_factor=2)
- **Output**: 80×80×512

**Math**:
- Width/Height: `40 × 2 = 80`
- Channels unchanged: 512

#### Step 14: Concat
- **Input1**: 80×80×512 (from upsample)
- **Input2**: 80×80×256 (from step 4, P3)
- **Operation**: Concatenation
- **Output**: 80×80×768

**Math**:
- Width/Height: 80×80
- Channels: `512 + 256 = 768`

#### Step 15: C2f Block (P3)
- **Input**: 80×80×768
- **Operation**: C2f shortcut=False, n=3, c=d
- **Output**: 80×80×256

**Math**:
- Spatial dimensions: 80×80
- Channels: 256

### Downsampling Path

#### Step 16: Convolution
- **Input**: 80×80×256 (P3)
- **Operation**: Conv k=3, s=2, p=1
- **Output**: 40×40×256

**Math**:
- Width/Height: `(80 + 2×1 - 3) / 2 + 1 = 79/2 + 1 = 40.5 → 40`
- Channels: 256

#### Step 17: Concat
- **Input1**: 40×40×256 (from conv)
- **Input2**: 40×40×512 (from step 12, P4)
- **Operation**: Concatenation
- **Output**: 40×40×768

**Math**:
- Channels: `256 + 512 = 768`

#### Step 18: C2f Block (P4)
- **Input**: 40×40×768
- **Operation**: C2f shortcut=False, n=3, c=d
- **Output**: 40×40×512

**Math**:
- Spatial dimensions: 40×40
- Channels: 512

#### Step 19: Convolution
- **Input**: 40×40×512 (P4)
- **Operation**: Conv k=3, s=2, p=1
- **Output**: 20×20×512

**Math**:
- Width/Height: `(40 + 2×1 - 3) / 2 + 1 = 39/2 + 1 = 20.5 → 20`
- Channels: 512

#### Step 20: Concat
- **Input1**: 20×20×512 (from conv)
- **Input2**: 20×20×512 (from step 9, P5)
- **Operation**: Concatenation
- **Output**: 20×20×1024

**Math**:
- Channels: `512 + 512 = 1024`

#### Step 21: C2f Block (P5)
- **Input**: 20×20×1024
- **Operation**: C2f shortcut=False, n=3, c=d
- **Output**: 20×20×512

**Math**:
- Spatial dimensions: 20×20
- Channels: 512

### Detection Heads

The final detection heads process the three feature maps:
- **P3**: 80×80×256 → Detect (small objects)
- **P4**: 40×40×512 → Detect (medium objects)  
- **P5**: 20×20×512 → Detect (large objects)

## Key Mathematical Principles

### 1. Convolution Output Size Formula
```
output_size = floor((input_size + 2×padding - kernel_size) / stride) + 1
```

### 2. Pooling/Upsampling
- **Max Pooling**: Reduces spatial dimensions by pooling factor
- **Upsampling**: Increases spatial dimensions by scale factor

### 3. Concatenation
- Spatial dimensions must match
- Channel dimensions are summed
- `output_channels = input1_channels + input2_channels`

### 4. Feature Pyramid Network (FPN) Logic
- **Bottom-up**: Progressive downsampling (2×, 4×, 8×, 16×, 32× reduction)
- **Top-down**: Upsampling with skip connections
- **Lateral connections**: Concatenation of features at same resolution

### 5. Multi-scale Detection
- **P3 (80×80)**: Detects small objects (high resolution, fine details)
- **P4 (40×40)**: Detects medium objects (balanced resolution/receptive field)
- **P5 (20×20)**: Detects large objects (low resolution, large receptive field)
