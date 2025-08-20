# ORB_SLAM3_DepthEnhancement

## Overview

This project presents an enhanced version of ORB-SLAM3 that incorporates real-time depth map refinement for RGB-D cameras. By implementing temporal fusion and spatial refinement techniques, we significantly improve the quality of depth measurements, leading to more robust tracking and mapping performance. The system is optimized for edge devices like Jetson Nano while maintaining real-time performance.

## Project Motivation

RGB-D cameras often suffer from several limitations that affect SLAM performance:

- **Depth measurement noise**: Especially prevalent in low-texture regions and at object boundaries
- **Missing depth values**: Holes in depth maps due to reflective surfaces, occlusions, or sensor limitations
- **Limited range**: Reliable depth measurements typically restricted to 0.5-4.5m
- **Edge bleeding**: Depth discontinuities at object boundaries

These issues can significantly degrade SLAM performance, causing:
- Failed feature tracking due to inconsistent depth values
- Inaccurate 3D point triangulation
- Drift in pose estimation
- Sparse or noisy 3D reconstructions

Our depth enhancement module addresses these challenges by leveraging temporal consistency and spatial coherence to produce more reliable depth estimates.

## Key Features

### 1. **Temporal Depth Fusion**
- Maintains a sliding window of historical depth frames
- Projects previous depth measurements to current frame using pose estimates
- Weights historical observations based on temporal proximity and confidence
- Reduces temporal noise through multi-frame integration

### 2. **Spatial Refinement**
- Focuses enhancement around ORB feature points where accurate depth is critical
- Applies edge-preserving filters to maintain depth discontinuities
- Removes outliers using local consistency checks
- Fills small holes using neighboring depth values

### 3. **Efficient 3D Point Pool Management**
- Maintains a sparse set of reliable 3D points across frames
- Uses spatial hashing for fast correspondence searches
- Automatically prunes old observations to limit memory usage
- Provides confidence estimates for each 3D point

### 4. **Jetson Nano Optimization**
- Downsampled processing (every 2nd pixel) for real-time performance
- Limited history buffer (5 frames) to reduce memory footprint
- Efficient data structures using spatial hashing
- Processing time: ~15-25ms per frame on Jetson Nano

### 5. **Comprehensive Evaluation Framework**
- Real-time metrics collection (CSV output)
- Measures valid pixel increase, hole filling, noise reduction
- Tracks processing time per frame
- Compares enhanced vs. original depth quality

## System Architecture

```
enhanceDepth (총괄 지휘)
|
|──> updateFeatureConfidenceMap
|      // 현재 프레임의 특징점(features)을 기반으로 mFeatureConfidenceMap (C_current) 생성
|
|──> temporalFusion (핵심 융합 로직)
|      |
|      └──> (과거 프레임 수만큼 반복)
|            |
|            |──> projectDepthToFrame
|            |      // 과거 Depth 맵을 현재 시점으로 변환 -> projectedDepth (D_past) 생성
|            |
|            └──> findCorrespondences
|                   // 현재 픽셀이 m3DPointPool에 있는지 조회 -> confidenceMap (C_past 역할) 생성
|
|──> spatialRefinement
|      // 특징점 주변의 로컬 노이즈 제거
|
|──> updateHistory
|      // 다음 프레임을 위해 현재 프레임 정보(향상된 depth 포함)를 mDepthHistory에 저장
|
└──> update3DPointPool (if KeyFrame)
       // 향상된 depth를 기반으로 m3DPointPool 업데이트
```

## Performance Improvements

Based on evaluation on TUM RGB-D dataset:

| Metric | Improvement | Description |
|--------|-------------|-------------|
| Valid Pixels | +15-25% | Percentage of pixels with valid depth measurements |
| Hole Filling | 500-2000 pixels/frame | Number of missing depth values recovered |
| Noise Reduction | 10-20% | Reduction in depth measurement standard deviation |
| Edge Preservation | 95%+ | Maintains depth discontinuities at object boundaries |
| Processing Time | 15-25ms | Additional processing time per frame |

## Requirements

### Hardware
- **Minimum**: NVIDIA Jetson Nano (4GB)
- **Recommended**: NVIDIA Jetson Xavier or desktop GPU
- RGB-D camera (Intel RealSense, Microsoft Kinect, ASUS Xtion)

### Software Dependencies
```bash
# Base requirements (same as ORB-SLAM3)
- Ubuntu 18.04 or 20.04
- OpenCV 4.2+
- Eigen3
- Pangolin
- DBoW2 and g2o (included in Thirdparty)

# Additional requirements
- C++14 compatible compiler
- CMake 3.10+
```

## Building the Project

### 1. Clone the Repository
```bash
git clone https://github.com/hyunseokkkk/ORB_SLAM3_DepthEnhancement
cd ORB_SLAM3_DepthEnhancement
```

### 2. Install Dependencies
```bash
# Install base dependencies
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    cmake \
    git \
    libeigen3-dev \
    libopencv-dev \
    libgl1-mesa-dev \
    libglew-dev

# Build Pangolin
git clone --recursive https://github.com/stevenlovegrove/Pangolin.git
cd Pangolin
mkdir build && cd build
cmake ..
make -j
sudo make install
```

### 3. Build Depth-Enhanced ORB-SLAM3
```bash
cd ~/ORB_SLAM3_DepthEnhancement
chmod +x build.sh
./build.sh
```

## Usage

### Running with Intel RealSense
```bash
./Examples/RGB-D/rgbd_realsense_D435i \
    Vocabulary/ORBvoc.txt \
    Examples/RGB-D/RealSense_D435i.yaml
```

### Configuration Parameters

Edit the configuration in `src/Tracking.cc`:
```cpp
// Depth enhancement parameters
config.maxHistorySize = 5;      // Number of frames in temporal buffer
config.maxKeyFrames = 3;        // Number of keyframes to maintain
config.patchSize = 3;           // Spatial filter patch size
config.processingStep = 2;      // Downsampling factor
config.outlierThreshold = 0.3f; // Depth difference threshold (meters)
config.minDepth = 0.1f;         // Minimum valid depth (meters)
config.maxDepth = 10.0f;        // Maximum valid depth (meters)
```

## Evaluation Results

### Quantitative Results on TUM RGB-D Dataset

#### fr1/desk Sequence
| Metric | Original | Enhanced | Improvement |
|--------|----------|----------|-------------|
| Valid Pixels | 73.2% | 91.5% | +18.3% |
| Depth Noise (std) | 0.024m | 0.019m | -20.8% |
| Holes per Frame | 2847 | 423 | -85.1% |
| Tracking Success | 92.3% | 97.8% | +5.5% |

#### fr2/xyz Sequence
| Metric | Original | Enhanced | Improvement |
|--------|----------|----------|-------------|
| Valid Pixels | 68.5% | 89.2% | +20.7% |
| Depth Noise (std) | 0.031m | 0.023m | -25.8% |
| Holes per Frame | 3564 | 687 | -80.7% |
| Tracking Success | 89.1% | 95.6% | +6.5% |

### Processing Time Analysis
```
Average processing time breakdown (Jetson Nano):
- Temporal Fusion: 8.3ms
- Spatial Refinement: 5.7ms
- 3D Point Pool Update: 2.1ms
- Total Overhead: 16.1ms
```

### Visual Comparison

<img width="173" height="144" alt="image" src="https://github.com/user-attachments/assets/854cb496-8c6e-49b0-bd12-643c1ce3a5ac" />
<img width="173" height="144" alt="image" src="https://github.com/user-attachments/assets/d6b566b4-437b-4ba8-8527-4c2958fce8b4" />

*Top: Original noisy depth map with holes. bottom: Enhanced depth map with filled holes and reduced noise.*

## Comparison with Original ORB-SLAM3

### Key Differences
1. **Tracking Robustness**: 5-10% improvement in challenging scenarios
2. **Mapping Density**: 20-30% more map points due to better depth estimates
3. **Trajectory Accuracy**: 10-15% reduction in ATE (Absolute Trajectory Error)
4. **Runtime Overhead**: 15-25ms additional processing per frame

## Advanced Usage

### Real-time Visualization
The system provides real-time visualization of:
- Enhanced depth maps
- Feature tracking
- 3D point cloud
- Camera trajectory

Enable visualization in the configuration file:
```yaml
Viewer.UseViewer: 1
Viewer.ViewpointX: 0
Viewer.ViewpointY: -10
Viewer.ViewpointZ: -0.1
Viewer.ViewpointF: 2000
```

## Troubleshooting

### Common Issues

1. **Low frame rate on Jetson Nano**
   - Increase `processingStep` to 3 or 4
   - Reduce `maxHistorySize` to 3
   - Disable visualization

2. **Memory issues**
   - Reduce 3D point pool size limit
   - Decrease history buffer size
   - Enable swap memory on Jetson

3. **Poor depth enhancement**
   - Check camera calibration
   - Adjust `outlierThreshold` based on sensor noise
   - Ensure sufficient lighting for RGB features

## Future Work

- [ ] GPU acceleration using CUDA
- [ ] Deep learning-based depth completion
- [ ] Multi-sensor fusion (RGB-D + IMU + LiDAR)
- [ ] Dynamic object filtering
- [ ] Semantic depth enhancement

## Acknowledgments

- Original ORB-SLAM3 by Carlos Campos, Richard Elvira, et al.
