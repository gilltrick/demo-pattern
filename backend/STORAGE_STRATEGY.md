<!--
Copyright (c) 2025 Gilltrick Patmann (gilltrick@gilltrick.de)

This file is part of AI Development Pattern Library.

This project is dual-licensed:
- For non-commercial use: Polyform Noncommercial License 1.0.0
- For commercial use: See LICENSE-COMMERCIAL

For full license details, see the LICENSE file in the root directory.
-->

# Storage Strategy
## S3-Compatible Object Storage Pattern

> **For AI:** File storage, signed URLs, CDN integration for scalable content delivery.

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)

---

## 🎯 Quick Reference

**Storage:** AWS S3 or MinIO (S3-compatible)

**Use Cases:** User uploads, processed media, static assets

**Security:** Pre-signed URLs, bucket policies, encryption at rest

**Performance:** CDN integration (CloudFront/CloudFlare)

---

## 📦 Storage Service Implementation

```typescript
// src/storage/storage.service.ts
import { Injectable } from '@nestjs/common'
import { S3 } from 'aws-sdk'
import { ConfigService } from '@nestjs/config'
import * as fs from 'fs'

@Injectable()
export class StorageService {
  private s3: S3

  constructor(private configService: ConfigService) {
    this.s3 = new S3({
      endpoint: this.configService.get('storage.endpoint'),  // MinIO or S3
      accessKeyId: this.configService.get('storage.accessKey'),
      secretAccessKey: this.configService.get('storage.secretKey'),
      s3ForcePathStyle: true,  // Required for MinIO
      signatureVersion: 'v4',
    })
  }

  async uploadFile(localPath: string, s3Key: string): Promise<string> {
    const bucket = this.configService.get('storage.bucket')
    const fileContent = fs.readFileSync(localPath)

    await this.s3.putObject({
      Bucket: bucket,
      Key: s3Key,
      Body: fileContent,
      ContentType: this.getContentType(s3Key),
    }).promise()

    return s3Key
  }

  async downloadFile(s3Key: string, localPath: string): Promise<void> {
    const bucket = this.configService.get('storage.bucket')

    const { Body } = await this.s3.getObject({
      Bucket: bucket,
      Key: s3Key,
    }).promise()

    fs.writeFileSync(localPath, Body as Buffer)
  }

  async getSignedUrl(s3Key: string, expiresIn: number = 3600): Promise<string> {
    const bucket = this.configService.get('storage.bucket')

    return this.s3.getSignedUrlPromise('getObject', {
      Bucket: bucket,
      Key: s3Key,
      Expires: expiresIn,  // seconds
    })
  }

  async getUploadUrl(s3Key: string, contentType: string): Promise<string> {
    const bucket = this.configService.get('storage.bucket')

    return this.s3.getSignedUrlPromise('putObject', {
      Bucket: bucket,
      Key: s3Key,
      ContentType: contentType,
      Expires: 300,  // 5 minutes to complete upload
    })
  }

  async deleteFile(s3Key: string): Promise<void> {
    const bucket = this.configService.get('storage.bucket')

    await this.s3.deleteObject({
      Bucket: bucket,
      Key: s3Key,
    }).promise()
  }

  private getContentType(filename: string): string {
    const ext = filename.split('.').pop()?.toLowerCase()
    const types = {
      'jpg': 'image/jpeg',
      'jpeg': 'image/jpeg',
      'png': 'image/png',
      'gif': 'image/gif',
      'mp4': 'video/mp4',
      'webm': 'video/webm',
      'pdf': 'application/pdf',
    }
    return types[ext] || 'application/octet-stream'
  }
}
```

---

## 🔐 Direct Upload Pattern (Client → S3)

**Why:** Avoid routing large files through backend (saves bandwidth, faster)

### Backend: Generate Upload URL

```typescript
// src/upload/upload.controller.ts
@Controller('upload')
export class UploadController {
  constructor(private storageService: StorageService) {}

  @Post('presigned-url')
  @UseGuards(JwtAuthGuard)
  async getUploadUrl(
    @Body() dto: { filename: string; contentType: string },
    @Request() req,
  ): Promise<{ uploadUrl: string; fileKey: string }> {
    const userId = req.user.userId
    const fileKey = `uploads/${userId}/${Date.now()}-${dto.filename}`

    const uploadUrl = await this.storageService.getUploadUrl(
      fileKey,
      dto.contentType
    )

    return { uploadUrl, fileKey }
  }
}
```

### Frontend: Direct Upload

```typescript
// Frontend code
async function uploadFile(file: File) {
  // 1. Get pre-signed URL from backend
  const { uploadUrl, fileKey } = await fetch('/api/upload/presigned-url', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      filename: file.name,
      contentType: file.type,
    }),
  }).then(r => r.json())

  // 2. Upload directly to S3
  await fetch(uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': file.type },
    body: file,
  })

  // 3. Notify backend upload is complete
  await fetch('/api/upload/complete', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ fileKey }),
  })
}
```

---

## 📥 Backend Upload Pattern

For smaller files or server-side processing:

```typescript
@Post('upload')
@UseGuards(JwtAuthGuard)
@UseInterceptors(FileInterceptor('file'))
async uploadFile(
  @UploadedFile() file: Express.Multer.File,
  @Request() req,
): Promise<{ fileKey: string; url: string }> {
  const userId = req.user.userId
  const fileKey = `uploads/${userId}/${Date.now()}-${file.originalname}`

  // Upload to S3
  await this.storageService.uploadFile(file.path, fileKey)

  // Get public URL or signed URL
  const url = await this.storageService.getSignedUrl(fileKey, 86400)  // 24 hours

  // Cleanup temp file
  fs.unlinkSync(file.path)

  return { fileKey, url }
}
```

---

## 🌐 CDN Integration

For public assets (videos, images):

```typescript
// src/config/configuration.ts
export default () => ({
  cdn: {
    enabled: process.env.CDN_ENABLED === 'true',
    domain: process.env.CDN_DOMAIN,  // e.g., https://cdn.example.com
  },
})

// src/storage/storage.service.ts
async getPublicUrl(s3Key: string): Promise<string> {
  const cdnEnabled = this.configService.get('cdn.enabled')
  const cdnDomain = this.configService.get('cdn.domain')

  if (cdnEnabled && cdnDomain) {
    return `${cdnDomain}/${s3Key}`
  }

  // Fallback to S3 direct URL
  return `https://${this.configService.get('storage.bucket')}.s3.amazonaws.com/${s3Key}`
}
```

---

## 🔒 Security Best Practices

### Bucket Policy (Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::ACCOUNT_ID:user/video-service" },
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/videos/*"
    }
  ]
}
```

### Server-Side Encryption

```typescript
await this.s3.putObject({
  Bucket: bucket,
  Key: s3Key,
  Body: fileContent,
  ServerSideEncryption: 'AES256',  // Encrypt at rest
}).promise()
```

---

## ✅ Implementation Checklist

- [ ] Configure S3 or MinIO credentials in Vault
- [ ] Create StorageService with upload/download methods
- [ ] Implement pre-signed URL generation
- [ ] Add direct upload endpoint (client → S3)
- [ ] Configure bucket policies (least privilege)
- [ ] Enable server-side encryption
- [ ] Set up CDN if serving public content
- [ ] Add file type validation
- [ ] Implement file size limits
- [ ] Add cleanup for failed uploads
- [ ] Monitor storage usage and costs

---

## 🔗 Related Patterns

→ [PROCESSING_PATTERNS.md](./PROCESSING_PATTERNS.md) - Download/upload for processing
→ [OBSERVABILITY.md](./OBSERVABILITY.md) - Monitor upload failures

---

**Pattern Version:** 1.0 (Demo)
**Last Updated:** 2025-01-09

← Back to [PATTERNS_INDEX.md](../getting-started/PATTERNS_INDEX.md)
