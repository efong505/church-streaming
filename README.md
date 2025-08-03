6# AWS Elemental MediaLive and MediaPackage Multi-Camera Streaming Guide

This guide updates the setup for streaming a church service using AWS Elemental MediaLive, MediaPackage, CloudFront, and S3, with a multi-camera setup (three cameras: pulpit, choir, congregation) managed via OBS Studio and RTMP outputs to YouTube and Facebook Live.

## Prerequisites
- **AWS Account**: Sign up at https://aws.amazon.com.
- **IAM Permissions**: Create an IAM user/role with permissions for MediaLive, MediaPackage, CloudFront, and S3 (use policies like `AWSMediaLiveFullAccess`, `AWSMediaPackageFullAccess`, `CloudFrontFullAccess`, `AmazonS3FullAccess`).
- **Hardware/Software**:
  - Three 1080p cameras (e.g., DSLR, webcam, or IP cameras) connected via HDMI/USB or RTMP (for IP cameras).
  - Microphones and audio mixer for clear audio.
  - Computer with OBS Studio (https://obsproject.com/) for encoding and scene switching.
  - Stable internet with 5–10 Mbps upload speed (wired Ethernet).
- **Church Website**: For embedding the HLS stream (e.g., S3 static hosting or WordPress).
- **Social Media Accounts**: YouTube and Facebook accounts with live streaming enabled. Obtain RTMP URLs and stream keys from:
  - YouTube: Studio > Create > Go Live > Stream Settings.
  - Facebook: Live Producer > Use a Persistent Stream Key.

## Step 1: Set Up Amazon S3 Bucket for VOD Storage
1. **Log in to AWS Management Console**:
   - Navigate to **S3**.
2. **Create a Bucket**:
   - Name: `church-service-vod-<your-account-id>` (unique).
   - Region: US East (N. Virginia).
   - Enable versioning and AES-256 encryption.
   - Block public access (use CloudFront for access).
3. **Create a Folder**:
   - Create a folder named `recordings`.

## Step 2: Configure Multi-Camera Setup in OBS Studio
1. **Install OBS Studio**:
   - Download from https://obsproject.com/.
2. **Add Camera Sources**:
   - Open OBS Studio.
   - Create three scenes (e.g., “Pulpit”, “Choir”, “Congregation”):
     - In **Sources**, click **+** > **Video Capture Device** for each camera (e.g., via HDMI/USB capture cards).
     - For IP cameras, use **Media Source** with RTMP URLs (e.g., `rtsp://<camera-ip>/live`).
   - Add an **Audio Input Capture** source for the microphone/mixer.
3. **Configure Scene Switching**:
   - Use OBS’s **Scene Collection** to switch between cameras manually or set up **Auto Scene Switcher** plugin for timed transitions (e.g., 30 seconds per scene).
   - Alternatively, use a hardware switcher (e.g., Blackmagic ATEM) to feed a single stream to OBS.
4. **Configure Output**:
   - Go to **Settings** > **Output**:
     - Output Mode: Advanced.
     - Encoder: x264 (or NVENC for GPU).
     - Bitrate: 5,000 Kbps (1080p).
     - Keyframe interval: 2 seconds.
   - Go to **Settings** > **Video**:
     - Output Resolution: 1920x1080.
     - Frame Rate: 30 fps.

## Step 3: Configure AWS Elemental MediaLive
1. **Navigate to MediaLive**:
   - Go to **AWS Elemental MediaLive**.
2. **Create an Input**:
   - Click **Inputs** > **Create input**.
   - **Input type**: RTMP (Push).
   - **Input name**: `ChurchServiceMultiCamInput`.
   - **Input class**: Single-input (or Standard for redundancy, +$0.588/hour).
   - **Input security group**: Allow RTMP traffic (port 1935) from your encoder’s IP.
   - Click **Create**. Note the **RTMP URL** and **Stream Key** (e.g., `rtmp://<endpoint>/live/<stream-key>`).
3. **Configure OBS for MediaLive**:
   - In OBS, go to **Settings** > **Stream**:
     - Service: Custom.
     - Server: Paste the MediaLive RTMP URL.
     - Stream Key: Paste the MediaLive Stream Key.
4. **Create a Channel**:
   - Click **Channels** > **Create channel**.
   - **Channel name**: `ChurchServiceMultiCamChannel`.
   - **Channel class**: Single-pipeline (cost-saving) or Standard (redundant).
   - **Input specification**:
     - Codec: H.264.
     - Resolution: 1080p.
     - Max bitrate: 5 Mbps.
   - **Input attachments**: Select `ChurchServiceMultiCamInput`.
   - **Encoder settings**:
     - **Video**:
       - Codec: H.264.
       - Output resolutions: 1080p (5 Mbps), 720p (2 Mbps), 576p (1.2 Mbps), 432p (0.8 Mbps), 288p (0.5 Mbps).
       - Frame rate: 30 fps.
     - **Audio**:
       - Codec: AAC.
       - Bitrate: 128 Kbps.
     - **Output groups**:
       - **HLS Group** (for website):
         - Destination: Create a new MediaPackage input (`ChurchServiceHLS`).
         - HLS settings: 5-second segments, 10 segments in playlist.
       - **RTMP Group (YouTube)**:
         - Destination: YouTube RTMP URL (e.g., `rtmp://a.rtmp.youtube.com/live2`).
         - Stream name: YouTube stream key.
       - **RTMP Group (Facebook)**:
         - Destination: Facebook RTMP URL (e.g., `rtmps://live-api-s.facebook.com:443/rtmp/`).
         - Stream name: Facebook stream key.
   - **General settings**: Enable captions (e.g., WebVTT) if needed.
   - Click **Create channel**. Note the channel ID.

## Step 4: Configure AWS Elemental MediaPackage
1. **Navigate to MediaPackage**:
   - Go to **AWS Elemental MediaPackage**.
2. **Create a Channel**:
   - Click **Channels** > **Create**.
   - **ID**: `ChurchServiceMultiCamChannel`.
   - **Input type**: HLS.
   - **Input URL**: Auto-populated from MediaLive HLS output.
   - Click **Create**.
3. **Create an Endpoint**:
   - In the channel, click **Add/Edit endpoints**.
   - **Endpoint ID**: `ChurchServiceHLS`.
   - **Packaging configuration**: HLS.
     - Segment duration: 5 seconds.
     - Playlist window: 60 seconds.
     - Enable DRM (optional).
   - **CDN configuration**: Select **Amazon CloudFront**.
   - Click **Save**. Note the **HLS Playback URL** (e.g., `https://<endpoint>.mediapackage.us-east-1.amazonaws.com/hls.m3u8`).

## Step 5: Configure Amazon CloudFront
1. **Navigate to CloudFront**:
   - Go to **CloudFront**.
2. **Create a Distribution**:
   - **Origin**:
     - Origin domain: Select the MediaPackage endpoint URL.
     - Protocol: HTTPS only.
   - **Default cache behavior**:
     - Viewer protocol policy: Redirect HTTP to HTTPS.
     - Allowed HTTP methods: GET, HEAD.
     - Cache policy: `Managed-CachingOptimized`.
   - **Price class**: All edge locations.
   - Click **Create distribution**. Note the **Distribution Domain Name** (e.g., `https://<distribution-id>.cloudfront.net`).

## Step 6: Embed the Stream on the Church Website
1. **Create an HTML Page**:
   - Use Video.js for HLS playback:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Church Service Live Stream</title>
  <link href="https://vjs.zencdn.net/7.20.3/video-js.css" rel="stylesheet" />
</head>
<body>
  <video-js id="church_stream" class="vjs-default-skin" controls preload="auto" width="640" height="360">
    <source src="https://<distribution-id>.cloudfront.net/hls.m3u8" type="application/x-mpegURL" />
  </video-js>
  <script src="https://vjs.zencdn.net/7.20.3/video.min.js"></script>
  <script>
    var player = videojs('church_stream');
  </script>
</body>
</html>
```

2. **Host the Page**:
   - Upload to an S3 bucket with static website hosting or integrate with your website (e.g., WordPress).

## Step 7: Archive to S3
1. **Enable Recording in MediaPackage**:
   - In the MediaPackage channel, go to **Recording configuration**.
   - Select the S3 bucket (`church-service-vod-<your-account-id>/recordings`).
   - Set recording duration (e.g., 1 hour).
2. **Access Recordings**:
   - Recordings appear in S3 as HLS segments. Use MediaConvert to transcode to MP4 for VOD.

## Step 8: Start and Test the Stream
1. **Start Streaming**:
   - In OBS, select scenes (Pulpit, Choir, Congregation) and click **Start Streaming**.
   - In MediaLive, start the channel (`ChurchServiceMultiCamChannel`).
2. **Test Playback**:
   - Verify the stream on the website, YouTube, and Facebook Live.
   - Check scene transitions and audio quality across devices.
3. **Monitor**:
   - Use MediaLive’s **Monitoring** tab and CloudWatch for metrics.

## Step 9: Stop Resources
1. **Stop Streaming**:
   - Stop OBS and the MediaLive channel to avoid idle costs ($0.588/hour).
2. **Clean Up**:
   - Delete unused channels/inputs and old S3 recordings.

## Cost Estimate (1,000–4,000 Viewers)
- **MediaLive**: $0.588/hour (input) + $0.0724/hour × 5 outputs = $0.95/hour × 4 hours = $3.80/month.
- **MediaPackage Ingest**: 4.175 GB/hour × 4 hours × $0.030 = $0.501/month.
- **MediaPackage Origination**: 1,000 viewers = 52.72 GB × $0.050 = $2.636/month; 4,000 viewers = 210.88 GB × $0.050 = $10.544/month.
- **CloudFront**: 1,000 viewers = 5,272 GB × $0.085 = $448.12/month; 4,000 viewers = 21,088 GB × $0.085 = $1,792.48/month.
- **S3**: 16.7 GB × $0.023 = $0.384/month.
- **Total**:
  - 1,000 viewers: $3.80 + $0.501 + $2.636 + $448.12 + $0.384 = **$455.44/month**.
  - 4,000 viewers: $3.80 + $0.501 + $10.544 + $1,792.48 + $0.384 = **$1,807.71/month**.
- **Note**: Multi-camera and RTMP outputs don’t increase costs (same bitrate). Reserved pricing saves ~70%.

## Resources
- AWS MediaLive: https://docs.aws.amazon.com/medialive/
- AWS MediaPackage: https://docs.aws.amazon.com/mediapackage/
- OBS Studio: https://obsproject.com/wiki/
- YouTube Live: https://support.google.com/youtube/answer/2474026
- Facebook Live: https://www.facebook.com/help/1536332383324684
