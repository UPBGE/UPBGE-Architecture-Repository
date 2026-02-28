# UPBGE VideoTexture Module Optimization Specification

**Project:** UPBGE (Unified Physics Blender Game Engine)  
**Module:** VideoTexture (gameengine/VideoTexture)  
**Version:** 1.0  
**Date:** February 19, 2026  
**Status:** In Progress - Block 1 Complete

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Project Context](#project-context)
3. [Technical Analysis](#technical-analysis)
4. [Implementation Plan](#implementation-plan)
5. [Block 1: Critical Bug Fixes](#block-1-critical-bug-fixes)
6. [Block 2: macOS AVFoundation Support](#block-2-macos-avfoundation-support)
7. [Block 3: NPOT Texture Support](#block-3-npot-texture-support)
8. [Block 4: Asynchronous GPU Upload (PBO)](#block-4-asynchronous-gpu-upload-pbo)
9. [Block 5: Audio Synchronization](#block-5-audio-synchronization)
10. [Testing Strategy](#testing-strategy)
11. [Performance Expectations](#performance-expectations)

---

## Executive Summary

This specification outlines a comprehensive optimization plan for UPBGE's VideoTexture module, focusing on performance improvements, platform support expansion, and critical bug fixes. The work is divided into five sequential implementation blocks.

**Primary Goals:**
- Fix critical bugs affecting stability and performance
- Add macOS camera/screen capture support via AVFoundation
- Enable NPOT (Non-Power-of-Two) texture support for modern GPUs
- Implement asynchronous CPU→GPU uploads using Pixel Buffer Objects
- Add audio synchronization using Audaspace integration

**Performance Targets:**
- Reduce HD video (1920×1080 RGBA) upload time from 8-12ms to 1-3ms per frame
- Eliminate unnecessary power-of-two scaling (quality improvement + VRAM savings)
- Remove dynamic_cast from per-frame hot path (CPU savings)
- Enable seamless audio/video sync for cutscenes and videomapping

---

## Project Context

### Background

UPBGE's VideoTexture module handles real-time video capture, playback, and texture manipulation for both game development and professional videomapping applications. The current implementation has several bottlenecks and limitations:

1. **Performance bottleneck:** Synchronous CPU→GPU texture uploads block the game loop
2. **Legacy restrictions:** Forced power-of-two texture scaling on modern hardware
3. **Platform gaps:** No native macOS camera capture support
4. **Missing features:** No audio synchronization for video sources
5. **Stability issues:** Several critical bugs affecting reliability

### Use Cases

1. **Game Development:** Video cutscenes, UI elements, dynamic textures
2. **Professional Videomapping:** Real-time projection mapping with multiple video sources
3. **Live Capture:** Webcam/capture card integration for interactive installations
4. **Mixed Reality:** Video compositing with 3D rendered content

### Technical Constraints

- **Zero Blender Core Changes:** All modifications confined to `source/blender/gameengine/`
- **Performance First:** Game engine context requires frame-time optimization
- **Multi-Platform:** OpenGL, Vulkan, and Metal backend support via Blender's GPU abstraction
- **Backward Compatibility:** Existing Python scripts must continue working

---

## Technical Analysis

### Current Architecture

```
Video Sources:
├── VideoFFmpeg (files, streams, cameras - Linux/Windows only)
├── VideoDeckLink (professional capture cards - hardware-specific)
├── ImageBuff (CPU buffers)
├── ImageViewport (render-to-texture)
└── ImageRender (offscreen 3D rendering)

Processing Pipeline:
[Source] → [Decode/Convert] → [CPU Buffer] → [GPU Upload] → [Texture] → [Material]
            (FFmpeg/sws_scale)  (RGBA8)       (BLOCKING)     (POT scaled)
```

### Identified Issues

**Category 1: Critical Bugs**
- **1.1** Duplicate `PyArg_ParseTuple` in `ImageBuff.cpp` causing buffer corruption
- **1.2** `m_origGpuTex` memory leak in `Texture::Close()` on double-close
- **1.3** `dynamic_cast` per frame in hot path (CPU overhead)
- **1.4** No resolution validation in `VideoFFmpeg::openCam()` (crash risk)

**Category 2: Performance Bottlenecks**
- **2.1** Synchronous `GPU_texture_update()` blocks game loop (8MB/frame for HD)
- **2.2** Unnecessary POT scaling on modern GPUs (quality loss + VRAM waste)
- **2.3** `IMB_scale` on main thread for non-POT textures

**Category 3: Missing Features**
- **3.1** No macOS camera/screen capture (FFmpeg has `avfoundation` support)
- **3.2** No audio sync (audio/video drift over time)

---

## Implementation Plan

### Block Sequence

```
Block 1: Critical Bugs     [COMPLETED]
   ↓
Block 2: macOS Support     [NEXT]
   ↓
Block 3: NPOT Textures     (prerequisite for Block 4)
   ↓
Block 4: PBO Upload        (highest performance impact)
   ↓
Block 5: Audio Sync        (most complex, requires stable base)
```

### Affected Files by Block

| Block | Files Modified | Lines Changed (est.) |
|-------|---------------|---------------------|
| 1 | `ImageBuff.cpp`, `Texture.h`, `Texture.cpp`, `VideoFFmpeg.cpp` | ~30 |
| 2 | `VideoFFmpeg.cpp` | ~50 |
| 3 | `ImageBase.cpp`, `Texture.cpp` | ~80 |
| 4 | `Texture.h`, `Texture.cpp` | ~150 |
| 5 | `VideoFFmpeg.h`, `VideoFFmpeg.cpp`, `blendVideoTex.cpp` | ~400 |

---

## Block 1: Critical Bug Fixes

**Status:** ✅ COMPLETED  
**Files:** `ImageBuff.cpp`, `Texture.h`, `Texture.cpp`, `VideoFFmpeg.cpp`

### Changes Implemented

#### Fix 1.1: Duplicate PyArg_ParseTuple in ImageBuff::load()

**File:** `ImageBuff.cpp` (lines 244-254)

**Problem:**
```cpp
// OLD CODE - lines appeared twice
if (!PyArg_ParseTuple(args, "s*hh:load", &buffer, &width, &height)) {
    PyErr_SetString(PyExc_TypeError, "Expected...");
    return nullptr;
}
if (!PyArg_ParseTuple(args, "s*hh:load", &buffer, &width, &height)) {
    PyErr_SetString(PyExc_TypeError, "Expected...");
    return nullptr;
}
```

**Solution:**
```cpp
// NEW CODE - single parse
if (!PyArg_ParseTuple(args, "s*hh:load", &buffer, &width, &height)) {
    return nullptr;
}
```

**Impact:**
- Prevents potential buffer double-acquisition
- Fixes missing `PyBuffer_Release` on first failure path
- Cleaner error handling

---

#### Fix 1.2: Memory Leak in Texture::Close()

**File:** `Texture.cpp` (lines 111-113)

**Problem:**
```cpp
// OLD CODE
if (m_origGpuTex) {
    m_imgTexture->runtime->gputexture[TEXTARGET_2D][0] = m_origGpuTex;
}
// m_origGpuTex not cleared - second Close() would restore stale pointer
```

**Solution:**
```cpp
// NEW CODE
if (m_origGpuTex) {
    m_imgTexture->runtime->gputexture[TEXTARGET_2D][0] = m_origGpuTex;
    m_origGpuTex = nullptr;  // prevent double-restoration
}
```

**Impact:**
- Prevents double-restore crash on destructor + explicit close
- Safe idempotent Close() behavior

---

#### Fix 1.3: Cached ImageRender Type Check

**Files:** `Texture.h` (line 68), `Texture.cpp` (lines 61, 133, 142-146)

**Problem:**
```cpp
// OLD CODE - dynamic_cast EVERY FRAME in hot path
ImageRender *imr = dynamic_cast<ImageRender *>(m_source ? m_source->m_image : nullptr);
```

**Solution:**
```cpp
// Texture.h - new member
bool m_isImageRender;

// Texture.cpp - cache in SetSource() (called once)
m_isImageRender = (dynamic_cast<ImageRender *>(source->m_image) != nullptr);

// loadTexture() hot path - zero-cost static_cast
ImageRender *imr = m_isImageRender ?
                       static_cast<ImageRender *>(m_source->m_image) :
                       nullptr;
```

**Impact:**
- Removes RTTI overhead from per-frame rendering
- ~0.1-0.5ms savings with multiple active video textures
- Zero cost: single bool added to struct

---

#### Fix 1.4: Resolution Validation in openCam()

**File:** `VideoFFmpeg.cpp` (lines 675-689)

**Problem:**
No validation after `openStream()` succeeds. Some capture devices return 0×0 resolution when:
- No signal present
- Unsupported parameters
- Driver initialization failed

**Solution:**
```cpp
if (openStream(filename, inputFormat, &formatParams) != 0)
    return;

// Guard: verify the driver returned a valid resolution
if (m_codecCtx->width <= 0 || m_codecCtx->height <= 0) {
    printf("VideoFFmpeg: capture device returned invalid resolution %dx%d\n",
           m_codecCtx->width, m_codecCtx->height);
    avcodec_free_context(&m_codecCtx);
    m_codecCtx = nullptr;
    avformat_close_input(&m_formatCtx);
    m_formatCtx = nullptr;
    return;
}
```

**Impact:**
- Prevents crashes from 0×0 buffer allocation
- Clean error reporting to console
- Proper resource cleanup on failure

---

### Testing

**Test Suite:** `test_bloque1.py`

**Coverage:**
- ✅ ImageBuff.load() with invalid arguments
- ✅ ImageBuff.load() with wrong buffer size
- ✅ ImageBuff.load() with correct buffer
- ✅ Texture.close() called twice (double-close safety)
- ✅ Texture source assignment (ImageBuff)
- ✅ Texture source change (cache update)
- ✅ VideoFFmpeg invalid camera device
- ✅ VideoFFmpeg file playback regression
- ✅ General regression tests (filters, plot, clear)

**Test Execution:**
```python
import test_bloque1
test_bloque1.run_all()
```

---

## Block 2: macOS AVFoundation Support

**Status:** 🔄 NEXT  
**Files:** `VideoFFmpeg.cpp`

### Objective

Add native macOS camera and screen capture support using FFmpeg's `avfoundation` input format. Currently, macOS users have no way to capture from cameras or screen within UPBGE.

### Technical Design

**Current State (Linux/Windows only):**
```cpp
#ifdef WIN32
    inputFormat = av_find_input_format("dshow");
    sprintf(filename, "video=%s", file);
#else  // Linux
    inputFormat = av_find_input_format("video4linux2");
    sprintf(filename, "/dev/video%d", camIdx);
#endif
```

**Proposed Change:**
```cpp
#ifdef WIN32
    inputFormat = av_find_input_format("dshow");
    sprintf(filename, "video=%s", file);
#elif defined(__APPLE__)
    inputFormat = av_find_input_format("avfoundation");
    // Device string format:
    // "0" = default camera
    // "1:" = screen capture (capture_screen format)
    sprintf(filename, "%d", camIdx);
    if (file && strstr(file, "screen")) {
        sprintf(filename, "%d:", camIdx);  // screen capture
    }
#else  // Linux
    inputFormat = av_find_input_format("video4linux2");
    sprintf(filename, "/dev/video%d", camIdx);
#endif
```

### Device String Format (macOS)

| Input | Meaning | FFmpeg Device String |
|-------|---------|---------------------|
| `VideoFFmpeg("", 0)` | Default camera | `"0"` |
| `VideoFFmpeg("", 1)` | Second camera | `"1"` |
| `VideoFFmpeg("screen", 1)` | Screen capture (display 1) | `"1:"` |

### Python API Example

```python
import VideoTexture

# Camera capture
cam = VideoTexture.VideoFFmpeg("", 0, width=1280, height=720)
cam.play()

# Screen capture
screen = VideoTexture.VideoFFmpeg("screen", 1)
screen.play()
```

### Testing Requirements

- Test on macOS with built-in camera
- Test with external USB camera
- Test screen capture on single/dual monitor setup
- Verify framerate and resolution parameters work correctly

---

## Block 3: NPOT Texture Support

**Status:** 📋 PLANNED  
**Files:** `ImageBase.cpp`, `Texture.cpp`

### Objective

Remove forced power-of-two texture scaling. Modern GPUs (OpenGL 2.0+, all Vulkan/Metal) support NPOT textures natively. Current behavior:
- 1920×1080 video → scaled to 2048×1024 (quality loss)
- Extra VRAM usage
- Unnecessary CPU scaling overhead

### Changes Required

#### Change 3.1: ImageBase::calcSize()

**File:** `ImageBase.cpp`

**Current:**
```cpp
void ImageBase::calcSize(short width, short height) {
    // Force power-of-two
    m_size[0] = 1;
    while (m_size[0] < width) m_size[0] <<= 1;
    m_size[1] = 1;
    while (m_size[1] < height) m_size[1] <<= 1;
}
```

**Proposed:**
```cpp
void ImageBase::calcSize(short width, short height) {
    // Use native resolution (all modern GPUs support NPOT)
    m_size[0] = width;
    m_size[1] = height;
}
```

---

#### Change 3.2: Texture::refresh() - Remove IMB_scale

**File:** `Texture.cpp`

**Current:**
```cpp
if (0) {  // Branch disabled but code present
    // Direct NPOT path - never executed
} else {
    // POT path - always executes
    ImBuf *ibuf = IMB_allocFromBuffer(...);
    m_scaledImBuf = IMB_scale(ibuf, ...);  // CPU scaling on main thread
    IMB_freeImBuf(ibuf);
}
```

**Proposed:**
```cpp
// Remove the if(0) check, use NPOT path directly
// No scaling needed - GPU handles NPOT textures natively
```

---

#### Change 3.3: DEG Update Guard

**File:** `Texture.cpp`

**Current:**
```cpp
// Called every frame even if texture didn't change
DEG_id_tag_update(&m_imgTexture->id, 0);
```

**Proposed:**
```cpp
// Only trigger depsgraph update if frame actually changed
if (m_avail) {
    DEG_id_tag_update(&m_imgTexture->id, 0);
}
```

### Performance Impact

| Scenario | Before | After | Improvement |
|----------|--------|-------|-------------|
| 1920×1080 RGBA upload | 2048×1024 (8.4MB) | 1920×1080 (7.9MB) | 6% less VRAM |
| CPU scaling time | 2-4ms/frame | 0ms | Eliminated |
| Visual quality | Scaled artifacts | Native resolution | Improved |

---

## Block 4: Asynchronous GPU Upload (PBO)

**Status:** 📋 PLANNED  
**Files:** `Texture.h`, `Texture.cpp`

### Objective

Implement double-buffered Pixel Buffer Objects for asynchronous CPU→GPU texture uploads. This is the **highest impact** performance optimization.

### Current Bottleneck

```
Frame N timeline:
CPU: [Decode video] [memcpy to texture buffer]
                                              ↓
GPU:                                    [STALLED - waiting for CPU]
                                              ↓
                                        [Upload 8MB synchronously]
                                              ↓
                                        [Render frame]
```

**Problem:** `GPU_texture_update()` is synchronous. GPU must wait for CPU to finish memcpy before it can proceed. For HD video (1920×1080 RGBA = 8MB), this takes 8-12ms, blocking the entire game loop.

### Proposed Solution: Double-Buffered PBO

```
Frame N timeline:
CPU: [Decode N] [Write to PBO[0]] ← non-blocking
GPU:            [Upload from PBO[1]] [Render frame N-1] ← parallel!

Frame N+1 timeline:
CPU: [Decode N+1] [Write to PBO[1]] ← non-blocking
GPU:              [Upload from PBO[0]] [Render frame N] ← parallel!
```

**Key:** CPU writes to one buffer while GPU reads from the other. No synchronization point.

### Implementation Design

#### Data Members (Texture.h)

```cpp
class Texture {
    // ... existing members ...
    
    // Double-buffer pixel buffers for async CPU→GPU upload
    GPUPixelBuffer *m_pixelBuffers[2];
    int             m_pixelBufferIndex;   // 0 or 1, alternates each frame
    size_t          m_pixelBufferSize;    // for detecting resolution changes
    bool            m_usePixelBuffer;     // enable only for video/capture sources
};
```

#### Texture::loadTexture() Logic

```cpp
void Texture::loadTexture(unsigned int *texture, short *size, ...) {
    if (!m_usePixelBuffer) {
        // Fallback for static images - synchronous upload
        GPU_texture_update(m_modifiedGPUTexture, GPU_DATA_UBYTE, texture);
        return;
    }
    
    int current  = m_pixelBufferIndex;
    int previous = 1 - m_pixelBufferIndex;
    
    // Recreate buffers if resolution changed
    size_t needed = size[0] * size[1] * 4;
    if (m_pixelBufferSize != needed) {
        // Free old buffers and create new ones
        if (m_pixelBuffers[0]) GPU_pixel_buffer_free(m_pixelBuffers[0]);
        if (m_pixelBuffers[1]) GPU_pixel_buffer_free(m_pixelBuffers[1]);
        m_pixelBuffers[0] = GPU_pixel_buffer_create(needed);
        m_pixelBuffers[1] = GPU_pixel_buffer_create(needed);
        m_pixelBufferSize = needed;
        
        // First frame: sync upload only
        GPU_texture_update(m_modifiedGPUTexture, GPU_DATA_UBYTE, texture);
        m_pixelBufferIndex = 0;
        return;
    }
    
    // CPU writes to current buffer (non-blocking)
    void *buf = GPU_pixel_buffer_map(m_pixelBuffers[current]);
    memcpy(buf, texture, needed);
    GPU_pixel_buffer_unmap(m_pixelBuffers[current]);
    
    // GPU uploads from previous buffer (DMA, doesn't block CPU)
    GPU_texture_update_sub_from_pixel_buffer(
        m_modifiedGPUTexture,
        GPU_DATA_UBYTE,
        m_pixelBuffers[previous],
        0, 0, 0,
        size[0], size[1], 1
    );
    
    // Swap buffers for next frame
    m_pixelBufferIndex = 1 - m_pixelBufferIndex;
}
```

#### Cleanup (Texture::Close)

```cpp
void Texture::Close() {
    // ... existing cleanup ...
    
    for (int i = 0; i < 2; i++) {
        if (m_pixelBuffers[i]) {
            GPU_pixel_buffer_free(m_pixelBuffers[i]);
            m_pixelBuffers[i] = nullptr;
        }
    }
    m_pixelBufferSize = 0;
    m_usePixelBuffer = false;
}
```

### GPU API Used

**Blender GPU Abstraction** (`GPU_texture.hh` lines 1229-1297):

```cpp
GPUPixelBuffer *GPU_pixel_buffer_create(size_t byte_size);
void            GPU_pixel_buffer_free(GPUPixelBuffer *pixel_buf);
void           *GPU_pixel_buffer_map(GPUPixelBuffer *pixel_buf);
void            GPU_pixel_buffer_unmap(GPUPixelBuffer *pixel_buf);

void GPU_texture_update_sub_from_pixel_buffer(
    gpu::Texture *texture,
    eGPUDataFormat data_format,
    GPUPixelBuffer *pixel_buf,
    int offset_x, int offset_y, int offset_z,
    int width, int height, int depth
);
```

**Cross-Platform Support:**
- ✅ OpenGL: Uses `GL_PIXEL_UNPACK_BUFFER` (PBO)
- ✅ Vulkan: Uses staging buffers
- ✅ Metal: Uses `MTLBuffer` with shared memory

### Performance Expectations

| Resolution | Before (sync) | After (PBO) | Improvement |
|------------|--------------|-------------|-------------|
| 1280×720 RGBA | 4-6ms | 1-2ms | 60-70% faster |
| 1920×1080 RGBA | 8-12ms | 2-3ms | 70-75% faster |
| 3840×2160 RGBA (4K) | 32-45ms | 8-12ms | 70-75% faster |

**Note:** Frame latency increases by 1 frame (acceptable tradeoff for massive throughput gain).

### Activation Strategy

PBO enabled only for:
- `VideoFFmpeg` sources where `!m_isFile || m_isStreaming`
- Capture card sources (VideoDeckLink)

PBO disabled for:
- Static images (`ImageBuff`, `ImageFFmpeg` with `m_isImage=true`)
- Reason: No benefit for one-time uploads, avoids buffer allocation overhead

---

## Block 5: Audio Synchronization

**Status:** 📋 PLANNED  
**Files:** `VideoFFmpeg.h`, `VideoFFmpeg.cpp`, `blendVideoTex.cpp`

### Objective

Add audio playback and synchronization for video sources using UPBGE's Audaspace integration. Currently, video and audio must be managed separately, leading to inevitable drift over time.

### Problem Statement

**Current Behavior:**
```python
# User must manage audio separately
video = VideoTexture.VideoFFmpeg("cutscene.mp4")
video.play()

# Audio played via separate actuator - no sync guarantee
audio_actuator.startSound()  # Will drift over time
```

**Issues:**
- Audio and video clocks are independent
- Drift accumulates: ±100ms after 1 minute is common
- Seek operations break sync completely
- User has no API to tie them together

### Design Overview

**Three Sync Modes:**

1. **SYNC_NONE** (default, backward compatible)
   - Current behavior: video uses `BLI_time_now_seconds()`
   - No audio integration

2. **SYNC_AUDIO** (new)
   - VideoFFmpeg decodes and plays audio internally
   - Video clock follows audio clock (`AUD_Handle_getPosition()`)
   - One-line Python API: `video.audio = True`

3. **SYNC_ACTUATOR** (new)
   - Video follows external `KX_SoundActuator`
   - For cases where audio is managed elsewhere
   - Python API: `video.audioHandle = actuator`

### Technical Design

#### VideoFFmpeg.h - New Members

```cpp
#ifdef WITH_AUDASPACE
#include "AUD_C-API.h"
#include "AUD_Special.h"

class VideoFFmpeg : public VideoBase {
    // ... existing members ...
    
    // Audio synchronization
    int                 m_audioStream;      // index of audio stream in AVFormatContext
    AVCodecContext     *m_audioCodecCtx;    // audio decoder context
    AUD_Sound          *m_audSound;         // Audaspace sound handle
    AUD_Handle         *m_audHandle;        // playback handle (master clock)
    int                 m_syncMode;         // SYNC_NONE/SYNC_AUDIO/SYNC_ACTUATOR
    float               m_audioVolume;      // 0.0-1.0
    float               m_audioDelay;       // manual A/V offset in seconds
    PyObject           *m_audioActuator;    // external KX_SoundActuator, nullable
};
#endif
```

#### Cache Thread - Audio Packet Routing

**Current (audio packets discarded):**
```cpp
if (packet.stream_index == m_videoStream) {
    // enqueue video packet
} else {
    av_packet_unref(&packet);  // ← audio packets thrown away
}
```

**Proposed:**
```cpp
if (packet.stream_index == m_videoStream) {
    // enqueue video packet
}
#ifdef WITH_AUDASPACE
else if (packet.stream_index == m_audioStream) {
    // decode audio packet
    // feed PCM to Audaspace via AUD_Sound_bufferFactory
}
#endif
else {
    av_packet_unref(&packet);
}
```

#### VideoFFmpeg::calcImage() - Clock Selection

```cpp
void VideoFFmpeg::calcImage(unsigned int texId, double ts) {
    double actTime;
    
#ifdef WITH_AUDASPACE
    if (m_syncMode == SYNC_AUDIO && m_audHandle) {
        // Audio is master clock
        actTime = AUD_Handle_getPosition(m_audHandle) - m_audioDelay;
    }
    else if (m_syncMode == SYNC_ACTUATOR && m_audioActuator) {
        // External actuator is master clock
        actTime = getActuatorPosition(m_audioActuator) - m_audioDelay;
    }
    else
#endif
    {
        // Current behavior: independent video clock
        actTime = BLI_time_now_seconds() - m_startTime;
    }
    
    // ... rest of frame selection logic ...
}
```

#### Python API

**New Properties (blendVideoTex.cpp):**

```python
# Enable internal audio playback
video.audio = True          # type: bool, default False
video.audioVolume = 0.8     # type: float, 0.0-1.0, default 1.0
video.audioDelay = 0.05     # type: float, A/V offset in seconds, default 0.0

# Sync with external actuator
video.audioHandle = actuator  # type: KX_SoundActuator or None

# Query sync mode
video.syncMode              # type: int, read-only
# Values: VideoTexture.SYNC_NONE (0)
#         VideoTexture.SYNC_AUDIO (1)
#         VideoTexture.SYNC_ACTUATOR (2)
```

**Constants:**
```python
VideoTexture.SYNC_NONE = 0
VideoTexture.SYNC_AUDIO = 1
VideoTexture.SYNC_ACTUATOR = 2
```

### Python Usage Examples

#### Example 1: Internal Audio

```python
import bge
from bge import logic
import VideoTexture

# Get texture object
obj = logic.getCurrentScene().objects["Screen"]
matID = VideoTexture.materialID(obj, "MAScreenMat")
tex = VideoTexture.Texture(obj, matID)

# Video source with integrated audio
video = VideoTexture.VideoFFmpeg("//cutscene.mp4")
video.audio = True          # Enable audio playback
video.audioVolume = 0.9     # 90% volume
tex.source = video
video.play()                # Audio starts automatically with video
```

#### Example 2: External Actuator Sync

```python
import bge
from bge import logic
import VideoTexture

obj = logic.getCurrentScene().objects["Screen"]
audio_obj = logic.getCurrentScene().objects["AudioEmitter"]

# Setup video
matID = VideoTexture.materialID(obj, "MAScreenMat")
tex = VideoTexture.Texture(obj, matID)
video = VideoTexture.VideoFFmpeg("//scene_no_audio.mp4")

# Sync with external audio actuator
audio_actuator = audio_obj.actuators["SceneAudio"]
video.audioHandle = audio_actuator

tex.source = video
video.play()
audio_actuator.startSound()  # Video follows this clock
```

#### Example 3: Manual A/V Offset Correction

```python
# If audio consistently leads video by 50ms
video.audio = True
video.audioDelay = 0.05  # Delay audio by 50ms to compensate
```

### Implementation Complexity

**Estimated Effort:** 3-5 days

**Complexity Factors:**
- FFmpeg audio decoding is well-documented
- Audaspace C API is straightforward
- Main complexity: thread-safe packet routing in cache thread
- Testing requires various codecs and sample rates

### Audio Format Support

Through FFmpeg, supports all common formats:
- Codecs: AAC, MP3, Opus, Vorbis, PCM, FLAC
- Sample rates: 8kHz - 192kHz
- Channels: Mono, Stereo, 5.1, 7.1
- Containers: MP4, MKV, AVI, MOV, WebM

---

## Testing Strategy

### Unit Tests

Each block includes Python test suite:
- `test_bloque1.py` - Bug fixes validation
- `test_bloque2.py` - macOS capture (requires Mac hardware)
- `test_bloque3.py` - NPOT texture validation
- `test_bloque4.py` - PBO performance benchmarks
- `test_bloque5.py` - Audio sync accuracy tests

### Integration Tests

**Scenario 1: Video Cutscene**
```python
# 30-second 1080p video with audio
# Validate: no drift, smooth playback, proper cleanup
```

**Scenario 2: Multi-Source Videomapping**
```python
# 4 concurrent 720p video textures
# Validate: frame rate stability, no memory leaks
```

**Scenario 3: Live Capture**
```python
# Webcam → real-time processing → projection
# Validate: low latency, proper device enumeration
```

### Performance Benchmarks

**Metrics to Track:**
- Frame upload time (ms)
- Overall frame time (ms)
- Memory usage (MB)
- CPU usage (%)
- GPU usage (%)

**Test Scenarios:**
- 720p @ 30fps
- 1080p @ 60fps
- 4K @ 30fps
- Multiple simultaneous sources

---

## Performance Expectations

### Summary Table

| Optimization | Resolution | Before | After | Improvement |
|-------------|------------|--------|-------|-------------|
| **Block 1** (dynamic_cast removal) | Any | baseline | -0.2ms/frame | CPU savings |
| **Block 3** (NPOT) | 1920×1080 | 2048×1024 scaled | 1920×1080 native | Quality + VRAM |
| **Block 4** (PBO) | 1920×1080 | 8-12ms upload | 2-3ms upload | 70-75% faster |
| **Block 5** (Audio sync) | N/A | Manual sync | Automatic | User experience |

### Combined Impact

**Before All Optimizations:**
- 1080p video texture: ~15ms total frame budget
- Breakdown: 8ms upload + 4ms decode + 3ms render

**After All Optimizations:**
- 1080p video texture: ~8ms total frame budget
- Breakdown: 2ms upload + 4ms decode + 2ms render

**Result:** ~50% reduction in total frame time, enabling:
- 60fps video playback on mid-range hardware
- Multiple simultaneous HD video sources
- Headroom for game logic and rendering

---

## Appendices

### A. Glossary

- **NPOT:** Non-Power-of-Two (texture dimensions not constrained to 2^n)
- **PBO:** Pixel Buffer Object (OpenGL async upload mechanism)
- **POT:** Power-of-Two (texture dimensions are 2^n: 256, 512, 1024, etc.)
- **DMA:** Direct Memory Access (GPU-initiated transfer, no CPU involvement)
- **Audaspace:** UPBGE's audio engine (fork of original BGE audio system)
- **FFmpeg:** Multimedia framework for video/audio decode
- **AVFoundation:** Apple's media framework (macOS/iOS)

### B. References

- [UPBGE Documentation](https://upbge.org/docs)
- [Blender GPU Module](https://developer.blender.org/docs/features/gpu/)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
- [OpenGL PBO Tutorial](https://www.songho.ca/opengl/gl_pbo.html)

### C. File Structure

```
source/blender/gameengine/VideoTexture/
├── Common.h
├── DeckLink.cpp / .h
├── Exception.cpp / .h
├── FilterBase.cpp / .h
├── FilterBlueScreen.cpp / .h
├── FilterColor.cpp / .h
├── FilterNormal.cpp / .h
├── FilterSource.cpp / .h
├── ImageBase.cpp / .h
├── ImageBuff.cpp / .h       [Block 1]
├── ImageMix.cpp / .h
├── ImageRender.cpp / .h
├── ImageViewport.cpp / .h
├── Texture.cpp / .h         [Block 1, 3, 4]
├── VideoBase.cpp / .h
├── VideoDeckLink.cpp / .h
├── VideoFFmpeg.cpp / .h     [Block 1, 2, 5]
├── blendVideoTex.cpp        [Block 5]
└── PyTypeList.cpp / .h
```

---

## Document History

| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2026-02-19 | 1.0 | Initial specification | Development Team |

---

**End of Specification Document**
