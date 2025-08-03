# AWS Elemental MediaLive and MediaPackage Multi-Camera Streaming Guide

This guide updates the setup for streaming a 3.5-hour weekly church service (14 hours/month) using AWS Elemental MediaLive, MediaPackage, CloudFront, and S3, with a multi-camera setup (three cameras: pulpit, choir, congregation) managed via OBS Studio, RTMP outputs to YouTube, Facebook Live, and Rumble, and viewer count analytics via CloudWatch and API Gateway.

## Prerequisites
- **AWS Account**: Sign up at https://aws.amazon.com.
- **IAM Permissions**: Create an IAM role with permissions for MediaLive, MediaPackage, CloudFront, S3, CloudWatch, Lambda, and API Gateway (use policies like `AWSMediaLiveFullAccess`, `AWSMediaPackageFullAccess`, `CloudFrontFullAccess`, `AmazonS3FullAccess`, `CloudWatchFullAccess`, `AWSLambdaFullAccess`, `AmazonAPIGatewayFullAccess`).
- **Hardware/Software**:
  - Three 1080p cameras (e.g., DSLR, webcam, or IP cameras) connected via HDMI/USB or RTMP.
  - Microphones and audio mixer.
  - Computer with OBS Studio (https://obsproject.com/).
  - Stable internet (5–10 Mbps upload, wired Ethernet).
- **Church Website**: For embedding the HLS stream and viewer count display.
- **Social Media Accounts**:
  - YouTube and Facebook accounts with live streaming enabled (RTMP URLs and stream keys from Studio/Live Producer).
  - Rumble account with live streaming enabled (RTMP URL and key from Rumble’s Live Streaming dashboard).
- **OBS Multi-RTMP Plugin**: For testing (https://github.com/sorayuki/obs-multi-rtmp).

## Step 1: Set Up Amazon S3 Bucket for VOD Storage
1. **Log in to AWS Management Console**:
   - Navigate to **S3**.
2. **Create a Bucket**:
   - Name: `church-service-vod-<your-account-id>` (unique).
   - Region: US East (N. Virginia).
   - Enable versioning and AES-256 encryption.
   - Block public access.
3. **Create a Folder**:
   - Create a folder named `recordings`.

## Step 2: Configure Multi-Camera Setup in OBS Studio
1. **Install OBS Studio**:
   - Download from https://obsproject.com/.
2. **Add Camera Sources**:
   - Create three scenes (“Pulpit”, “Choir”, “Congregation”):
     - Add **Video Capture Device** for each camera (via HDMI/USB) or **Media Source** for IP cameras (e.g., `rtsp://<camera-ip>/live`).
     - Add **Audio Input Capture** for the microphone/mixer.
3. **Configure Scene Switching**:
   - Use OBS’s **Scene Collection** or **Auto Scene Switcher** plugin for transitions (e.g., 30 seconds per scene).
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
   - Click **Create**. Note the **RTMP URL** and **Stream Key**.
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
       - **RTMP Group (Rumble)**:
         - Destination: Rumble RTMP URL (e.g., `rtmp://ingest.rumble.com/live`).
         - Stream name: Rumble stream key (from Rumble’s Live Streaming dashboard).
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

## Step 6: Set Up Viewer Count Analytics
1. **Enable CloudWatch Metrics in MediaPackage**:
   - In MediaPackage, go to the channel > **Monitoring** > Enable **CloudWatch Metrics**.
   - Metrics like `ActiveSessions` and `EgressBytes` will track viewer activity [].
2. **Create a Lambda Function**:
   - Go to **Lambda** > **Create function**.
   - **Function name**: `ViewerCountAggregator`.
   - **Runtime**: Python 3.9.
   - **Code**:

```python
import json
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch')

def lambda_handler(event, context):
    metric_data = cloudwatch.get_metric_data(
        MetricDataQueries=[
            {
                'Id': 'viewerCount',
                'MetricStat': {
                    'Metric': {
                        'Namespace': 'AWS/MediaPackage',
                        'MetricName': 'ActiveSessions',
                        'Dimensions': [
                            {'Name': 'Channel', 'Value': 'ChurchServiceMultiCamChannel'},
                            {'Name': 'Endpoint', 'Value': 'ChurchServiceHLS'}
                        ]
                    },
                    'Period': 60,
                    'Stat': 'Average'
                }
            }
        ],
        StartTime=datetime.utcnow() - timedelta(minutes=5),
        EndTime=datetime.utcnow()
    )
    viewer_count = int(metric_data['MetricDataResults'][0]['Values'][0]) if metric_data['MetricDataResults'][0]['Values'] else 0
    return {
        'statusCode': 200,
        'body': json.dumps({'viewer_count': viewer_count}),
        'headers': {'Access-Control-Allow-Origin': '*'}
    }
```

   - Attach IAM role with `CloudWatchReadOnlyAccess` and `AWSLambdaBasicExecutionRole`.
   - Set a CloudWatch Events trigger (e.g., every 1 minute).

3. **Create API Gateway Endpoint**:
   - Go to **API Gateway** > **Create API** > REST API.
   - **API name**: `ViewerCountAPI`.
   - Create a **GET** method for `/viewers`.
   - Map to the Lambda function (`ViewerCountAggregator`).
   - Enable CORS for website integration.
   - Deploy to a stage (e.g., `prod`). Note the endpoint URL (e.g., `https://<api-id>.execute-api.us-east-1.amazonaws.com/prod/viewers`).

4. **Integrate with Website**:
   - Add JavaScript to your website to fetch viewer counts:

```html
<script>
  async function updateViewerCount() {
    const response = await fetch('https://<api-id>.execute-api.us-east-1.amazonaws.com/prod/viewers');
    const data = await response.json();
    document.getElementById('viewer-count').innerText = `Viewers: ${data.viewer_count}`;
  }
  setInterval(updateViewerCount, 60000); // Update every minute
</script>
<div id="viewer-count">Viewers: 0</div>
```

## Step 7: Embed the Stream on the Church Website
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
  <div id="viewer-count">Viewers: 0</div>
  <script src="https://vjs.zencdn.net/7.20.3/video.min.js"></script>
  <script>
    var player = videojs('church_stream');
    async function updateViewerCount() {
      const response = await fetch('https://<api-id>.execute-api.us-east-1.amazonaws.com/prod/viewers');
      const data = await response.json();
      document.getElementById('viewer-count').innerText = `Viewers: ${data.viewer_count}`;
    }
    setInterval(updateViewerCount, 60000);
  </script>
</body>
</html>
```

2. **Host the Page**:
   - Upload to an S3 bucket with static website hosting or integrate with your website (e.g., WordPress).

## Step 8: Archive to S3
1. **Enable Recording in MediaPackage**:
   - In the MediaPackage channel, go to **Recording configuration**.
   - Select the S3 bucket (`church-service-vod-<your-account-id>/recordings`).
   - Set recording duration (e.g., 3.5 hours).
2. **Access Recordings**:
   - Recordings appear in S3 as HLS segments. Use MediaConvert to transcode to MP4 for VOD.

## Step 9: Start and Test the Stream
1. **Start Streaming**:
   - In OBS, select scenes and click **Start Streaming**.
   - In MediaLive, start the channel (`ChurchServiceMultiCamChannel`).
2. **Test Playback**:
   - Verify the stream on the website, YouTube, Facebook Live, and Rumble.
   - Check scene transitions, audio quality, and viewer count updates.
3. **Monitor**:
   - Use MediaLive’s **Monitoring** tab, CloudWatch for viewer metrics, and API Gateway logs.

## Step 10: Stop Resources
1. **Stop Streaming**:
   - Stop OBS and the MediaLive channel to avoid idle costs ($0.588/hour).
2. **Clean Up**:
   - Delete unused channels/inputs and old S3 recordings.

## Cost Estimate (1,000–4,000 Viewers, 14 Hours/Month)
- **MediaLive**: $0.588/hour (input) + $0.0724/hour × 5 outputs = $0.95/hour × 14 hours = **$13.30/month**.
- **MediaPackage Ingest**: 4.175 GB/hour × 14 hours × $0.030 = **$1.754/month**.
- **MediaPackage Origination**: 1% of viewer data × $0.050/GB:
  - 1,000 viewers: 18.452 GB/viewer × 1,000 × 0.01 = 184.52 GB × $0.050 = **$9.226/month**.
  - 4,000 viewers: 18.452 GB/viewer × 4,000 × 0.01 = 738.08 GB × $0.050 = **$36.904/month**.
- **CloudFront**: Total viewer data × $0.085/GB (99% cache hit):
  - 1,000 viewers: 18.452 GB/viewer × 1,000 = 18,452 GB × $0.085 = **$1,568.42/month**.
  - 4,000 viewers: 18.452 GB/viewer × 4,000 = 73,808 GB × $0.085 = **$6,273.68/month**.
- **S3**: 58.45 GB × $0.023 = **$1.344/month**.
- **Lambda**: 1M requests/month (free tier, negligible cost).
- **API Gateway**: 1M requests/month × $3.50/M = **$3.50/month**.
- **Total**:
  - 1,000 viewers: $13.30 + $1.754 + $9.226 + $1,568.42 + $1.344 + $3.50 = **$1,597.54/month**.
  - 4,000 viewers: $13.30 + $1.754 + $36.904 + $6,273.68 + $1.344 + $3.50 = **$6,330.50/month**.

## Resources
- AWS MediaLive: https://docs.aws.amazon.com/medialive/
- AWS MediaPackage: https://docs.aws.amazon.com/mediapackage/
- OBS Studio: https://obsproject.com/wiki/
- YouTube Live: https://support.google.com/youtube/answer/2474026
- Facebook Live: https://www.facebook.com/help/1536332383324684
- Rumble Live: https://help.rumble.com/en/articles/1561686-go-live
- CloudWatch Metrics: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/
