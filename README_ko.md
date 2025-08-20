# ORB_SLAM3_DepthEnhancement

[English Version](./README.md)

---

## 개요

이 프로젝트는 RGB-D 카메라를 위한 실시간 깊이 맵 보정을 통합한 ORB-SLAM3의 향상된 버전을 제시합니다. 시간적 융합과 공간적 보정 기법을 적용하여 깊이 측정 품질을 크게 향상시킴으로써 추적과 매핑 성능을 더욱 견고하게 만듭니다. 본 시스템은 Jetson Nano와 같은 엣지 디바이스에서도 실시간 성능을 유지하도록 최적화되어 있습니다.

## 프로젝트 동기

RGB-D 카메라는 SLAM 성능에 영향을 주는 여러 한계를 가지고 있습니다:

- **깊이 측정 노이즈**: 저 텍스처 영역 및 객체 경계에서 특히 심각함  
- **깊이 값 누락**: 반사 표면, 가려짐, 센서 한계로 인한 깊이 맵의 구멍 발생  
- **제한된 거리 범위**: 신뢰할 수 있는 깊이 측정은 보통 0.5~4.5m 범위에 제한  
- **에지 블리딩**: 객체 경계에서 깊이 불연속성이 퍼짐  

이 문제들은 다음을 유발합니다:
- 불일치하는 깊이 값으로 인한 특징점 추적 실패  
- 부정확한 3D 포인트 삼각측량  
- 자세 추정 드리프트  
- 희소하거나 노이즈가 많은 3D 재구성  

본 깊이 보정 모듈은 시간적 일관성과 공간적 응집성을 활용하여 보다 신뢰할 수 있는 깊이 추정을 제공합니다.

## 주요 기능

### 1. **시간적 깊이 융합 (Temporal Fusion)**
- 과거 깊이 프레임을 저장하는 슬라이딩 윈도우 유지
- 자세 추정을 통해 과거 깊이를 현재 프레임으로 투영
- 시간적 근접성과 신뢰도를 기반으로 가중치 적용
- 다중 프레임 통합으로 시간적 노이즈 감소

### 2. **공간적 보정 (Spatial Refinement)**
- ORB 특징점 주변 깊이 품질 집중 향상
- 깊이 불연속성을 유지하는 에지 보존 필터 적용
- 로컬 일관성 검사를 통한 이상치 제거
- 주변 깊이 값을 활용한 작은 구멍 보정

### 3. **효율적인 3D 포인트 풀 관리**
- 프레임 간 신뢰성 있는 희소 3D 포인트 유지
- 공간 해싱을 이용한 빠른 대응 검색
- 메모리 사용 제한을 위해 오래된 관측치 자동 제거
- 각 3D 포인트별 신뢰도 제공

### 4. **Jetson Nano 최적화**
- 실시간 성능을 위해 2픽셀 단위 다운샘플링
- 메모리 사용을 줄이기 위한 5프레임 히스토리 버퍼
- 공간 해싱 기반 효율적 데이터 구조
- 처리 시간: Jetson Nano에서 프레임당 약 15~25ms

### 5. **종합적인 평가 프레임워크**
- 실시간 메트릭 수집 (CSV 출력)
- 유효 픽셀 증가, 구멍 보정, 노이즈 감소 측정
- 프레임별 처리 시간 기록
- 향상 전/후 깊이 품질 비교 제공

## 시스템 아키텍처

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

## 성능 향상

TUM RGB-D 데이터셋 평가 기준:

| Metric | Improvement | Description |
|--------|-------------|-------------|
| Valid Pixels | +15-25% | Percentage of pixels with valid depth measurements |
| Hole Filling | 500-2000 pixels/frame | Number of missing depth values recovered |
| Noise Reduction | 10-20% | Reduction in depth measurement standard deviation |
| Edge Preservation | 95%+ | Maintains depth discontinuities at object boundaries |
| Processing Time | 15-25ms | Additional processing time per frame |

## 요구사항

### 하드웨어
- **Minimum**: NVIDIA Jetson Nano (4GB)
- **Recommended**: NVIDIA Jetson Xavier or desktop GPU
- RGB-D camera (Intel RealSense, Microsoft Kinect, ASUS Xtion)

### 소프트웨어
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

## 빌드 방법

### 1. 저장소 클론
```bash
git clone https://github.com/hyunseokkkk/ORB_SLAM3_DepthEnhancement
cd ORB_SLAM3_DepthEnhancement
```

### 2. 의존성 설치
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
```
```
# RealSense SDK 
sudo apt-get install -y libssl-dev libusb-1.0-0-dev libx11-dev
git clone https://github.com/IntelRealSense/librealsense.git
cd librealsense
mkdir build && cd build
cmake .. -DBUILD_EXAMPLES=true
make -j
sudo make install
sudo ldconfig
cd ../..
```
```
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

## Intel Realsense를 이용한 실시간 실행
```bash
./Examples/RGB-D/rgbd_realsense_D435i \
    Vocabulary/ORBvoc.txt \
    Examples/RGB-D/RealSense_D435i.yaml
```

## 설정 파라미터

src/Tracking.cc에서 설정을 수정하세요:
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

## 성능 평가 결과

### TUM RGB-D 데이터셋 정량적 결과

#### fr1/desk 시퀀스
| Metric | Original | Enhanced | Improvement |
|--------|----------|----------|-------------|
| Valid Pixels | 73.2% | 91.5% | +18.3% |
| Depth Noise (std) | 0.024m | 0.019m | -20.8% |
| Holes per Frame | 2847 | 423 | -85.1% |
| Tracking Success | 92.3% | 97.8% | +5.5% |

#### fr2/xyz 시퀀스
| Metric | Original | Enhanced | Improvement |
|--------|----------|----------|-------------|
| Valid Pixels | 68.5% | 89.2% | +20.7% |
| Depth Noise (std) | 0.031m | 0.023m | -25.8% |
| Holes per Frame | 3564 | 687 | -80.7% |
| Tracking Success | 89.1% | 95.6% | +6.5% |

### 처리 시간 분석
```
평균 처리 시간 분석 (Jetson Nano 기준):
- Temporal Fusion: 8.3ms
- Spatial Refinement: 5.7ms
- 3D Point Pool Update: 2.1ms
- Total Overhead: 16.1ms
```

### 비주얼적 비교

<img width="173" height="144" alt="image" src="https://github.com/user-attachments/assets/854cb496-8c6e-49b0-bd12-643c1ce3a5ac" />
<img width="172" height="144" alt="image" src="https://github.com/user-attachments/assets/78c0aff5-a0a9-4fa0-9c48-65f3a5622cef" />


*왼쪽: Original noisy depth map with holes. 오른쪽: Enhanced depth map with filled holes and reduced noise.*

## 원본 ORB-SLAM3와의 비교

### 주요 차이점
1. **트래킹 강건성**: 5-10% improvement in challenging scenarios
2. **맵핑 밀도**: 20-30% more map points due to better depth estimates
3. **궤적 정확도**: 10-15% reduction in ATE (Absolute Trajectory Error)
4. **런타임 오버헤드**: 15-25ms additional processing per frame

## 고급 사용법

### 실시간 시각화
시스템은 다음 항목에 대한 실시간 시각화를 제공합니다:
- 향상된 Depth Map
- 특징점 추적
- 3D 포인트 클라우드
- 카메라 궤적

설정 파일에서 시각화를 활성화하세요:
```yaml
Viewer.UseViewer: 1
Viewer.ViewpointX: 0
Viewer.ViewpointY: -10
Viewer.ViewpointZ: -0.1
Viewer.ViewpointF: 2000
```

## 문제 해결

### 일반적인 문제

1. **Jetson Nano에서 낮은 프레임 속도**
   - `processingStep` to 3 or 4 증가
   - `maxHistorySize` to 3 감소
   - visualization 비활성화

2. **메모리 문제**
   - 3D 포인트 풀 크기 제한 감소
   - 히스토리 버퍼 크기 감소
   - Jetson에서 스왑 메모리 활성화

3. **Depth 향상 성능 저하**
   - 카메라 캘리브레이션 확인
   - 센서 노이즈에 따라  `outlierThreshold` 조정
   - RGB 특징점을 위해 충분한 조명 확보

## 향후 계획

- [ ] CUDA를 이용한 GPU 가속
- [ ] 딥러닝 기반 Depth 완성(completion)
- [ ] 다중 센서 융합 (RGB-D + IMU + LiDAR)
- [ ] 동적 객체 필터링
- [ ] 시맨틱 Depth 향상

## Acknowledgments

- Original ORB-SLAM3 by Carlos Campos, Richard Elvira, et al.


