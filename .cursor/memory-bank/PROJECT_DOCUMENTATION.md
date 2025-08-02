# Short Video Maker - Project Documentation

## Overview

Short Video Maker is an automated video creation tool that generates short-form video content (TikTok, Instagram Reels, YouTube Shorts) from simple text inputs. It combines text-to-speech, automatic captions, background videos, and music to create engaging videos without requiring heavy GPU resources or expensive third-party APIs.

## Project Architecture

### Core Components

The project is built with a modular architecture consisting of several key components:

1. **Server Layer** - Express.js server with REST API and MCP (Model Context Protocol) endpoints
2. **Video Processing Engine** - Core video creation logic
3. **External Libraries** - Integration with various AI/ML services
4. **Web UI** - React-based user interface
5. **Video Rendering** - Remotion-based video composition

## Project Flow

### 1. Application Startup (`src/index.ts`)

```typescript
// Main entry point
async function main() {
  // 1. Initialize configuration
  const config = new Config();

  // 2. Initialize music manager
  const musicManager = new MusicManager(config);

  // 3. Initialize core libraries
  const remotion = await Remotion.init(config);
  const kokoro = await Kokoro.init(config.kokoroModelPrecision);
  const whisper = await Whisper.init(config);
  const ffmpeg = await FFMpeg.init();
  const pexelsApi = new PexelsAPI(config.pexelsApiKey);

  // 4. Create video processing engine
  const shortCreator = new ShortCreator(
    config,
    remotion,
    kokoro,
    whisper,
    ffmpeg,
    pexelsApi,
    musicManager,
  );

  // 5. Start server
  const server = new Server(config, shortCreator);
  server.start();
}
```

### 2. Configuration Management (`src/config.ts`)

The `Config` class manages all environment variables and system settings:

- **API Keys**: Pexels API key for video sourcing
- **Model Settings**: Whisper model type, Kokoro precision
- **Performance**: Concurrency, cache sizes for Docker optimization
- **Paths**: Data directories, temporary files, music files
- **System**: Port, log levels, Docker detection

### 3. Video Creation Process (`src/short-creator/ShortCreator.ts`)

The core video generation follows this workflow:

#### 3.1 Queue Management

```typescript
public addToQueue(sceneInput: SceneInput[], config: RenderConfig): string {
  const id = cuid(); // Generate unique video ID
  this.queue.push({ sceneInput, config, id });
  if (this.queue.length === 1) {
    this.processQueue(); // Start processing if first item
  }
  return id;
}
```

#### 3.2 Scene Processing

For each scene in the video:

1. **Text-to-Speech Generation** (`Kokoro.ts`)

   ```typescript
   const audio = await this.kokoro.generate(scene.text, config.voice);
   // Returns: { audio: ArrayBuffer, audioLength: number }
   ```

2. **Caption Generation** (`Whisper.ts`)

   ```typescript
   await this.ffmpeg.saveNormalizedAudio(audioStream, tempWavPath);
   const captions = await this.whisper.CreateCaption(tempWavPath);
   // Returns: Caption[] with timing information
   ```

3. **Background Video Selection** (`Pexels.ts`)

   ```typescript
   const video = await this.pexelsApi.findVideo(
     scene.searchTerms,
     audioLength,
     excludeVideoIds,
     orientation,
   );
   // Returns: Video object with URL and metadata
   ```

4. **Audio Processing** (`FFmpeg.ts`)
   ```typescript
   await this.ffmpeg.saveToMp3(audioStream, tempMp3Path);
   ```

#### 3.3 Video Composition (`Remotion.ts`)

```typescript
await this.remotion.render(
  {
    music: selectedMusic,
    scenes: processedScenes,
    config: {
      durationMs: totalDuration * 1000,
      paddingBack: config.paddingBack,
      captionBackgroundColor: config.captionBackgroundColor,
      captionPosition: config.captionPosition,
      musicVolume: config.musicVolume,
    },
  },
  videoId,
  orientation,
);
```

### 4. External Library Integrations

#### 4.1 Kokoro TTS (`src/short-creator/libraries/Kokoro.ts`)

- **Purpose**: Text-to-speech generation
- **Model**: ONNX-based Kokoro-82M-v1.0
- **Features**:
  - 30+ voice options (male/female, different accents)
  - Multiple precision levels (fp32, fp16, q8, q4, q4f16)
  - Streaming audio generation
  - WAV buffer concatenation

#### 4.2 Whisper Speech Recognition (`src/short-creator/libraries/Whisper.ts`)

- **Purpose**: Generate accurate captions from audio
- **Model**: Whisper.cpp with configurable model sizes
- **Features**:
  - Token-level timestamps for precise caption timing
  - Multiple model options (tiny, base, small, medium, large)
  - Automatic installation and model downloading
  - Progress tracking during transcription

#### 4.3 Pexels API (`src/short-creator/libraries/Pexels.ts`)

- **Purpose**: Source background videos
- **Features**:
  - Search by keywords with fallback to joker terms
  - Orientation-specific video selection (portrait/landscape)
  - Duration matching with buffer time
  - Quality filtering (HD resolution)
  - Retry logic with timeout handling
  - Duplicate video exclusion

#### 4.4 FFmpeg Integration (`src/short-creator/libraries/FFmpeg.ts`)

- **Purpose**: Audio/video processing
- **Features**:
  - Audio normalization
  - Format conversion (WAV to MP3)
  - Audio buffer manipulation
  - Temporary file management

#### 4.5 Remotion Video Rendering (`src/short-creator/libraries/Remotion.ts`)

- **Purpose**: Programmatic video composition
- **Features**:
  - React-based video components
  - Multi-scene composition
  - Audio synchronization
  - Caption rendering with timing
  - Background music integration
  - Multiple orientation support

### 5. Server Architecture (`src/server/`)

#### 5.1 REST API (`src/server/routers/rest.ts`)

Endpoints:

- `POST /api/short-video` - Create new video
- `GET /api/short-video/{id}/status` - Check video status
- `GET /api/short-video/{id}` - Download completed video
- `GET /api/short-videos` - List all videos
- `DELETE /api/short-video/{id}` - Delete video
- `GET /api/voices` - List available voices
- `GET /api/music-tags` - List music moods
- `GET /api/tmp/{file}` - Serve temporary files
- `GET /api/music/{file}` - Serve music files

#### 5.2 MCP Server (`src/server/routers/mcp.ts`)

Model Context Protocol integration for AI agent compatibility:

- `create-short-video` - Create video via MCP
- `get-video-status` - Check video status via MCP

### 6. Web UI (`src/ui/`)

#### 6.1 React Application Structure

- **App.tsx**: Main router with three routes
- **VideoList**: Display all created videos
- **VideoCreator**: Form-based video creation interface
- **VideoDetails**: Video status and download page

#### 6.2 Video Creator Interface (`src/ui/pages/VideoCreator.tsx`)

Features:

- Dynamic scene addition/removal
- Text input for narration
- Search terms for background videos
- Configuration options:
  - Voice selection (30+ options)
  - Music mood selection (12 moods)
  - Caption position (top/center/bottom)
  - Caption background color
  - Video orientation (portrait/landscape)
  - Music volume control
  - End screen padding

### 7. Video Components (`src/components/`)

#### 7.1 Remotion Video Components

- **PortraitVideo.tsx**: 1080x1920 vertical video layout
- **LandscapeVideo.tsx**: 1920x1080 horizontal video layout
- **Test.tsx**: Simple test component for validation

#### 7.2 Video Composition Features

- **Multi-scene Support**: Sequential scene rendering
- **Audio Synchronization**: Perfect timing between audio and video
- **Caption Rendering**:
  - Token-level timing accuracy
  - Configurable positioning
  - Background highlighting
  - Font styling with Google Fonts
- **Background Music**: Loop with volume control
- **Video Backgrounds**: Offthread video loading for performance

### 8. Music Management (`src/short-creator/music.ts`)

#### 8.1 Music Library

- **30+ royalty-free tracks** from various artists
- **12 mood categories**: sad, melancholic, happy, euphoric, excited, chill, uneasy, angry, dark, hopeful, contemplative, funny
- **Duration mapping**: Each track has start/end timestamps
- **Automatic selection**: Random selection based on mood and video duration

### 9. Data Flow

#### 9.1 Video Creation Request Flow

```
1. User submits video creation request
   ↓
2. Request validated and added to queue
   ↓
3. For each scene:
   a. Generate TTS audio (Kokoro)
   b. Create captions (Whisper)
   c. Find background video (Pexels)
   d. Process audio files (FFmpeg)
   ↓
4. Select background music (MusicManager)
   ↓
5. Compose final video (Remotion)
   ↓
6. Save to disk and update status
```

#### 9.2 File Management

- **Temporary Files**: WAV, MP3, MP4 files during processing
- **Video Storage**: Final MP4 files in videos directory
- **Music Files**: Static music library in static/music
- **Model Files**: Whisper models downloaded to libs directory

### 10. Configuration Options

#### 10.1 Environment Variables

```bash
# Required
PEXELS_API_KEY=your_pexels_api_key

# Optional
LOG_LEVEL=info|debug|warn|error
PORT=3123
WHISPER_MODEL=medium.en
KOKORO_MODEL_PRECISION=fp32
CONCURRENCY=1
VIDEO_CACHE_SIZE_IN_BYTES=2097152000
```

#### 10.2 Video Configuration

```typescript
interface RenderConfig {
  paddingBack?: number; // End screen duration (ms)
  music?: MusicMoodEnum; // Background music mood
  captionPosition?: CaptionPositionEnum; // top/center/bottom
  captionBackgroundColor?: string; // CSS color
  voice?: VoiceEnum; // TTS voice
  orientation?: OrientationEnum; // portrait/landscape
  musicVolume?: MusicVolumeEnum; // muted/low/medium/high
}
```

### 11. Deployment Options

#### 11.1 Docker Images

- **tiny**: Lightweight with q4 Kokoro model, tiny.en Whisper
- **normal**: Standard with fp32 Kokoro model, base.en Whisper
- **cuda**: GPU-accelerated with medium.en Whisper model

#### 11.2 System Requirements

- **RAM**: ≥3GB (4GB recommended)
- **CPU**: ≥2 vCPUs
- **Storage**: ≥5GB
- **Platform**: Ubuntu ≥22.04, macOS (Windows not supported)

### 12. Integration Examples

#### 12.1 REST API Usage

```bash
# Create video
curl -X POST http://localhost:3123/api/short-video \
  -H "Content-Type: application/json" \
  -d '{
    "scenes": [{
      "text": "Hello world!",
      "searchTerms": ["nature", "landscape"]
    }],
    "config": {
      "paddingBack": 1500,
      "music": "chill",
      "voice": "af_heart"
    }
  }'

# Check status
curl http://localhost:3123/api/short-video/{videoId}/status

# Download video
curl http://localhost:3123/api/short-video/{videoId} -o video.mp4
```

#### 12.2 MCP Integration

The server exposes MCP endpoints for AI agent integration:

- `/mcp/sse` - Server-sent events endpoint
- `/mcp/messages` - Message handling endpoint

### 13. Performance Considerations

#### 13.1 Memory Management

- **Concurrency Control**: Limits parallel browser tabs during rendering
- **Video Cache**: Configurable cache size for offthread video frames
- **Temporary File Cleanup**: Automatic cleanup after video creation
- **Model Precision**: Configurable Kokoro model precision for memory/performance trade-off

#### 13.2 Processing Optimization

- **Queue System**: Sequential processing to prevent resource conflicts
- **Streaming Audio**: Kokoro generates audio in streams
- **Offthread Video**: Remotion loads videos off-thread for better performance
- **Progress Tracking**: Real-time progress updates during processing

### 14. Error Handling

#### 14.1 Graceful Degradation

- **Pexels API Failures**: Fallback to joker terms (nature, globe, space, ocean)
- **Video Download Failures**: Retry logic with exponential backoff
- **Model Loading**: Automatic installation and validation
- **File System**: Directory creation and permission handling

#### 14.2 Validation

- **Input Validation**: Zod schemas for all API inputs
- **File Validation**: Music file existence checks
- **Model Validation**: Whisper and Kokoro model verification
- **Configuration Validation**: Environment variable validation

### 15. Future Enhancements

#### 15.1 Potential Improvements

- **Multi-language Support**: Extend beyond English
- **Custom Video Sources**: Allow user-uploaded videos
- **Advanced Caption Styling**: More customization options
- **Video Templates**: Pre-defined video layouts
- **Batch Processing**: Multiple video creation
- **Real-time Preview**: Live video preview during creation

#### 15.2 Scalability Considerations

- **Horizontal Scaling**: Multiple server instances
- **Database Integration**: Persistent video metadata
- **Cloud Storage**: External video storage
- **CDN Integration**: Global video delivery
- **Load Balancing**: Request distribution

## Conclusion

The Short Video Maker project demonstrates a sophisticated approach to automated video creation by combining multiple AI/ML technologies in a modular, scalable architecture. The system efficiently processes text inputs through a pipeline of TTS, speech recognition, video sourcing, and composition to create professional-quality short-form videos suitable for social media platforms.

The project's strength lies in its:

- **Modular Design**: Easy to extend and maintain
- **Resource Efficiency**: CPU-based processing without heavy GPU requirements
- **Multiple Interfaces**: REST API, MCP, and Web UI
- **Comprehensive Configuration**: Extensive customization options
- **Robust Error Handling**: Graceful degradation and validation
- **Production Ready**: Docker deployment and monitoring capabilities

