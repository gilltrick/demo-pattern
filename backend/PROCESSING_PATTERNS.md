<!--
Copyright (c) 2025 Gilltrick Patmann (gilltrick@gilltrick.de)

This file is part of AI Development Pattern Library.

This project is dual-licensed:
- For non-commercial use: Polyform Noncommercial License 1.0.0
- For commercial use: See LICENSE-COMMERCIAL

For full license details, see the LICENSE file in the root directory.
-->

# Processing Patterns
## Event-Driven Media & Data Processing

> **For AI:** Patterns for building scalable, fault-tolerant processing services (video, images, files, data).

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)

---

## 🎯 Quick Reference

**Use Case:** Video encoding, image processing, file transformations, batch jobs

**Architecture:** Event-driven with queue-based processing

**Tools:**
- **FFmpeg** - Video/audio processing
- **Sharp** - Image processing
- **RabbitMQ** - Job queue
- **S3/MinIO** - File storage

**Reliability:** Retry logic, progress tracking, failure handling

---

## 🏗️ Architecture Overview

```
[Upload Service]
      ↓ (publishes event)
  video.uploaded
      ↓
[RabbitMQ Queue]
      ↓ (consumes)
[Processing Service]
      ↓ (downloads from S3)
  processes with FFmpeg
      ↓ (uploads result to S3)
  publishes video.processed
      ↓
[Notification Service] notifies user
```

**Benefits:**
- **Scalable** - Add more workers during high load
- **Resilient** - Failed jobs retry automatically
- **Async** - Upload service doesn't wait for processing

---

## 🎬 FFmpeg Integration Pattern

### Installation in Docker

```dockerfile
# Dockerfile
FROM node:18-alpine

# Install FFmpeg
RUN apk add --no-cache ffmpeg

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

CMD ["node", "dist/main"]
```

### Processing Service Implementation

```typescript
// src/processing/ffmpeg.service.ts
import { Injectable } from '@nestjs/common'
import { exec } from 'child_process'
import { promisify } from 'util'
import * as path from 'path'
import * as fs from 'fs/promises'

const execPromise = promisify(exec)

export interface TranscodeOptions {
  inputPath: string
  outputPath: string
  resolution?: '1080p' | '720p' | '480p' | '360p'
  codec?: 'h264' | 'h265'
  preset?: 'ultrafast' | 'fast' | 'medium' | 'slow'
}

@Injectable()
export class FFmpegService {
  async transcode(options: TranscodeOptions): Promise<void> {
    const resolutions = {
      '1080p': '1920x1080',
      '720p': '1280x720',
      '480p': '854x480',
      '360p': '640x360',
    }

    const resolution = resolutions[options.resolution || '720p']
    const codec = options.codec || 'h264'
    const preset = options.preset || 'fast'

    const command = [
      'ffmpeg',
      '-i', options.inputPath,
      '-vf', `scale=${resolution}`,
      '-c:v', codec === 'h265' ? 'libx265' : 'libx264',
      '-preset', preset,
      '-crf', '23',              // Quality (lower = better, 18-28 range)
      '-c:a', 'aac',             // Audio codec
      '-b:a', '128k',            // Audio bitrate
      '-movflags', '+faststart', // Enable streaming
      '-y',                      // Overwrite output
      options.outputPath,
    ].join(' ')

    try {
      await execPromise(command)
    } catch (error) {
      throw new Error(`FFmpeg transcoding failed: ${error.message}`)
    }
  }

  async extractThumbnail(videoPath: string, outputPath: string, timeSeconds: number = 1): Promise<void> {
    const command = [
      'ffmpeg',
      '-i', videoPath,
      '-ss', timeSeconds.toString(),  // Seek to timestamp
      '-vframes', '1',                 // Extract 1 frame
      '-q:v', '2',                     // Quality
      '-y',
      outputPath,
    ].join(' ')

    await execPromise(command)
  }

  async getVideoMetadata(videoPath: string): Promise<any> {
    const command = [
      'ffprobe',
      '-v', 'quiet',
      '-print_format', 'json',
      '-show_format',
      '-show_streams',
      videoPath,
    ].join(' ')

    const { stdout } = await execPromise(command)
    return JSON.parse(stdout)
  }
}
```

---

## 🎯 Event-Driven Processing Flow

### Step 1: Listen for Upload Events

```typescript
// src/events/video-events.controller.ts
import { Controller } from '@nestjs/common'
import { EventPattern } from '@nestjs/microservices'

@Controller()
export class VideoEventsController {
  constructor(
    private processingService: ProcessingService,
    private inboxService: InboxService,
  ) {}

  @EventPattern('video.uploaded')
  async handleVideoUploaded(event: DomainEvent): Promise<void> {
    // Idempotency check
    if (await this.inboxService.isProcessed(event.eventId)) {
      return
    }

    const { videoId, s3Key, userId } = event.payload

    try {
      // Queue processing job
      await this.processingService.queueTranscoding({
        videoId,
        s3Key,
        userId,
        resolutions: ['1080p', '720p', '480p'],
      })

      await this.inboxService.markProcessed(event.eventId, event.eventType)
    } catch (error) {
      console.error('Failed to queue transcoding job:', error)
      throw error  // Will retry
    }
  }
}
```

---

### Step 2: Process with Progress Tracking

```typescript
// src/processing/processing.service.ts
import { Injectable } from '@nestjs/common'

@Injectable()
export class ProcessingService {
  constructor(
    private ffmpegService: FFmpegService,
    private storageService: StorageService,
    private progressService: ProgressService,
  ) {}

  async processVideo(job: ProcessingJob): Promise<void> {
    const { videoId, s3Key, resolutions } = job

    try {
      // Update progress: Starting
      await this.progressService.update(videoId, {
        status: 'processing',
        progress: 0,
        message: 'Downloading video...',
      })

      // 1. Download from S3
      const inputPath = `/tmp/${videoId}-input.mp4`
      await this.storageService.downloadFile(s3Key, inputPath)

      await this.progressService.update(videoId, {
        progress: 10,
        message: 'Video downloaded, starting transcoding...',
      })

      // 2. Process each resolution
      const outputs = []
      for (let i = 0; i < resolutions.length; i++) {
        const resolution = resolutions[i]
        const outputPath = `/tmp/${videoId}-${resolution}.mp4`

        await this.progressService.update(videoId, {
          progress: 10 + (i / resolutions.length) * 70,
          message: `Transcoding ${resolution}...`,
        })

        // Transcode
        await this.ffmpegService.transcode({
          inputPath,
          outputPath,
          resolution,
        })

        // Upload to S3
        const s3OutputKey = `videos/${videoId}/${resolution}.mp4`
        await this.storageService.uploadFile(outputPath, s3OutputKey)

        outputs.push({ resolution, s3Key: s3OutputKey })

        // Cleanup temp file
        await fs.unlink(outputPath)
      }

      // 3. Extract thumbnail
      await this.progressService.update(videoId, {
        progress: 90,
        message: 'Generating thumbnail...',
      })

      const thumbnailPath = `/tmp/${videoId}-thumbnail.jpg`
      await this.ffmpegService.extractThumbnail(inputPath, thumbnailPath)

      const thumbnailKey = `thumbnails/${videoId}.jpg`
      await this.storageService.uploadFile(thumbnailPath, thumbnailKey)

      // Cleanup
      await fs.unlink(inputPath)
      await fs.unlink(thumbnailPath)

      // 4. Publish completion event
      await this.progressService.update(videoId, {
        status: 'completed',
        progress: 100,
        message: 'Processing complete!',
      })

      await this.publishEvent('video.processing.completed', {
        videoId,
        outputs,
        thumbnail: thumbnailKey,
      })

    } catch (error) {
      await this.progressService.update(videoId, {
        status: 'failed',
        progress: 0,
        message: `Processing failed: ${error.message}`,
      })

      throw error  // Retry logic will handle
    }
  }
}
```

---

### Step 3: Progress Tracking (Real-Time Updates)

```typescript
// src/progress/progress.service.ts
import { Injectable } from '@nestjs/common'
import { InjectRepository } from '@nestjs/typeorm'
import { Repository } from 'typeorm'

@Entity()
export class ProcessingProgress {
  @PrimaryColumn()
  videoId: string

  @Column()
  status: 'queued' | 'processing' | 'completed' | 'failed'

  @Column({ type: 'int', default: 0 })
  progress: number  // 0-100

  @Column({ nullable: true })
  message: string

  @Column({ type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  updatedAt: Date
}

@Injectable()
export class ProgressService {
  constructor(
    @InjectRepository(ProcessingProgress)
    private progressRepo: Repository<ProcessingProgress>,
  ) {}

  async update(videoId: string, data: Partial<ProcessingProgress>): Promise<void> {
    await this.progressRepo.upsert(
      { videoId, ...data, updatedAt: new Date() },
      ['videoId']
    )

    // Optional: Publish progress event for real-time updates
    await this.publishEvent('video.progress.updated', {
      videoId,
      ...data,
    })
  }

  async getProgress(videoId: string): Promise<ProcessingProgress> {
    return this.progressRepo.findOne({ where: { videoId } })
  }
}
```

---

## 🔄 Retry & Error Handling

```typescript
// src/processing/processing.consumer.ts
import { Controller } from '@nestjs/common'
import { MessagePattern, Ctx, RmqContext } from '@nestjs/microservices'

@Controller()
export class ProcessingConsumer {
  private readonly maxRetries = 3

  @MessagePattern('video.process')
  async processVideo(@Payload() job: ProcessingJob, @Ctx() context: RmqContext): Promise<void> {
    const channel = context.getChannelRef()
    const originalMsg = context.getMessage()

    try {
      await this.processingService.processVideo(job)

      // Success - acknowledge message
      channel.ack(originalMsg)

    } catch (error) {
      const retryCount = originalMsg.properties.headers['x-retry-count'] || 0

      if (retryCount < this.maxRetries) {
        // Retry - requeue with incremented counter
        channel.nack(originalMsg, false, false)  // Don't requeue (we'll republish)

        await this.publishWithRetry(job, retryCount + 1)
      } else {
        // Max retries reached - send to DLQ
        channel.ack(originalMsg)
        await this.sendToDeadLetterQueue(job, error)
      }
    }
  }

  private async publishWithRetry(job: ProcessingJob, retryCount: number): Promise<void> {
    // Exponential backoff delay
    const delay = Math.min(1000 * Math.pow(2, retryCount), 30000)

    setTimeout(() => {
      this.rabbitClient.emit('video.process', job, {
        headers: { 'x-retry-count': retryCount },
      })
    }, delay)
  }
}
```

---

## ✅ Implementation Checklist

- [ ] Install FFmpeg in Docker image
- [ ] Create FFmpegService with transcode/thumbnail methods
- [ ] Set up event listener for upload events
- [ ] Implement inbox pattern for idempotency
- [ ] Create processing service with progress tracking
- [ ] Add retry logic with exponential backoff
- [ ] Configure dead letter queue for failures
- [ ] Implement progress tracking table/API
- [ ] Add temp file cleanup
- [ ] Test with various video formats
- [ ] Monitor processing queue depth
- [ ] Set up alerts for failed jobs

---

## 🔗 Related Patterns

→ [STORAGE_STRATEGY.md](./STORAGE_STRATEGY.md) - S3 integration for file storage
→ [MESSAGING_STRATEGY.md](./MESSAGING_STRATEGY.md) - Event publishing and consumption
→ [OBSERVABILITY.md](./OBSERVABILITY.md) - Monitoring processing jobs

---

## 🎓 How AI Uses This Pattern

When you request: **"Build a video processing service"**

**AI will:**
1. Read this pattern
2. Create FFmpegService with transcode/thumbnail methods
3. Set up event-driven architecture for async processing
4. Implement progress tracking
5. Add retry logic with exponential backoff
6. Integrate with storage service (S3)
7. Include all error handling
8. Add monitoring and alerts

**Result:** Production-ready video processor in 2-3 hours

---

**Pattern Version:** 1.0 (Demo)
**Last Updated:** 2025-01-09
**Status:** Active

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)
