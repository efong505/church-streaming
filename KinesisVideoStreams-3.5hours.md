# AWS Kinesis Video Streams Multi-Camera Streaming Guide

This guide updates the setup for streaming a 3.5-hour weekly church service (14 hours/month) using Amazon Kinesis Video Streams, CloudFront, and S3, with a multi-camera setup (three cameras: pulpit, choir, congregation) managed via OBS Studio, WebRTC for low-latency viewers, Restream for YouTube/Facebook/Rumble, and viewer count analytics via CloudWatch and API Gateway.

## Prerequisites
- **AWS Account**: Sign up at https://aws.amazon.com.
- **IAM Permissions**: Create an IAM role with permissions for Kinesis Video Streams, CloudFront, S3, CloudWatch, Lambda, and API Gateway (use policies like `AmazonKinesisVideoStreamsFullAccess`, `CloudFrontFullAccess`, `AmazonS3FullAccess`, `CloudWatchFullAccess`, `AWSLambdaFullAccess`, `AmazonAPIGatewayFullAccess`).
- **Hardware/Software**:
  - Three 1080p cameras (e.g., DSLR, webcam, or IP cameras).
  - Microphones and audio mixer.
  - Computer with OBS Studio (https://obsproject.com/) and multi-RTMP plugin (https://github.com/sorayuki/obs-multi-rtmp).
  - Stable internet (5–10 Mbps upload).
- **Church Website**: For embedding HLS/WebRTC streams and viewer count display.
- **Restream Account**: Sign up at https://restream.io/ for YouTube, Facebook, and Rumble streaming (RTMP ingest URL and key).
- **Social Media Accounts**: YouTube, Facebook, and Rumble accounts with live streaming enabled.

## Step 1: Set Up Amazon S3 Bucket
1. **Log in to AWS Console**:
   - Navigate to **S3**.
2. **Create a Bucket**:
   - Name: `church-service-vod-<your-account-id>` (unique).
   - Region: US East (N. Virginia).
   - Enable versioning and AES-256 encryption.
   - Block public access.
3. **Create a Folder**:
   - Create a folder named `recordings`.

## Step 2: Configure Multi-Camera Setup in OBS Studio
1. **Install OBS Studio and Multi-RTMP Plugin**:
   - Download OBS from https://obsproject.com/.
   - Install multi-RTMP plugin from https://github.com/sorayuki/obs-multi-rtmp.
2. **Add Camera Sources**:
   - Create three scenes (“Pulpit”, “Choir”, “Congregation”):
     - Add **Video Capture Device** for each camera or **Media Source** for IP cameras (e.g., `rtsp://<camera-ip>/live`).
     - Add **Audio Input Capture** for the microphone/mixer.
3. **Configure Scene Switching**:
   - Use OBS’s **Scene Collection** or **Auto Scene Switcher** plugin.
4. **Configure Output for Kinesis and Restream**:
   - Go to **Settings** > **Stream**:
     - Service: Custom.
     - Server: `rtmp://<ingest-endpoint>.kinesisvideo.us-east-1.amazonaws.com:1935/app`.
     - Stream Key: `ChurchServiceStream`.
   - Add multi-RTMP output:
     - Service: Custom.
     - Server: Restream RTMP URL (e.g., `rtmp://live.restream.io/live`).
     - Stream Key: Restream stream key (configure YouTube, Facebook, Rumble in Restream dashboard).
   - Go to **Settings** > **Output**:
     - Encoder: x264.
     - Bitrate: 5,000 Kbps.
     - Keyframe interval: 2 seconds.
   - Go to **Settings** > **Video**:
     - Output Resolution: 1920x1080.
     - Frame Rate: 30 fps.

## Step 3: Configure Kinesis Video Streams
1. **Navigate to Kinesis Video Streams**:
   - Go to **Kinesis** > **Video Streams**.
2. **Create a Stream**:
   - **Stream name**: `ChurchServiceStream`.
   - **Data retention**: 24 hours.
   - Click **Create video stream**. Note the **Ingest Endpoint** and **Stream ARN**.
3. **Enable WebRTC**:
   - Go to **Signaling Channels** > **Create signaling channel**.
   - Name: `ChurchServiceWebRTC`.
   - Note the **Channel ARN**.

## Step 4: Configure Social Media Streaming with Restream
1. **Set Up Restream**:
   - Sign up at https://restream.io/.
   - Add YouTube, Facebook, and Rumble as destinations in Restream Studio.
   - Copy the Restream RTMP URL and stream key.
2. **Configure OBS for Restream**:
   - In OBS, use the multi-RTMP plugin to add the Restream output (configured above).

## Step 5: Configure Amazon CloudFront for HLS Delivery
1. **Navigate to CloudFront**:
   - Go to **CloudFront**.
2. **Create a Distribution**:
   - **Origin**: Kinesis HLS URL (e.g., `https://<endpoint>.kinesisvideo.us-east-1.amazonaws.com/hls/v1/getHLSMasterPlaylist.m3u8?StreamName=ChurchServiceStream`).
   - **Protocol**: HTTPS only.
   - **Cache behavior**: `Managed-CachingOptimized`.
   - **Price class**: All edge locations.
   - Click **Create distribution**. Note the **Distribution Domain Name**.

## Step 6: Set Up Viewer Count Analytics
1. **Enable CloudWatch Metrics in Kinesis**:
   - In Kinesis Video Streams, enable metrics for `GetHLSStreamingSessionURL` (HLS) and `GetSignalingChannelEndpoint` (WebRTC) [].
2. **Create a Lambda Function**:
   - Go to **Lambda** > **Create function**.
   - **Function name**: `KinesisViewerCountAggregator`.
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
                        'Namespace': 'AWS/KinesisVideo',
                        'MetricName': 'GetHLSStreamingSessionURL',
                        'Dimensions': [
                            {'Name': 'StreamName', 'Value': 'ChurchServiceStream'}
                        ]
                    },
                    'Period': 60,
                    'Stat': 'Sum'
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
   - **API name**: `KinesisViewerCountAPI`.
   - Create a **GET** method for `/viewers`.
   - Map to the Lambda function (`KinesisViewerCountAggregator`).
   - Enable CORS.
   - Deploy to a stage (e.g., `prod`). Note the endpoint URL.

4. **Integrate with Website**:
   - Add JavaScript to fetch viewer counts (see Step 7).

## Step 7: Embed Streams on the Church Website
1. **HLS Playback (Standard Latency)**:
   - Use Video.js for HLS:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Church Service Live Stream (HLS)</title>
  <link href="https://vjs.zencdn.net/7.20.3/video-js.css" rel="stylesheet" />
</head>
<body>
  <video-js id="church_stream" class="vjs-default-skin" controls preload="auto" width="640" height="360">
    <source src="https://<distribution-id>.cloudfront.net/hls/v1/getHLSMasterPlaylist.m3u8" type="application/x-mpegURL" />
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

2. **WebRTC Playback (Low Latency)**:
   - Use Kinesis WebRTC SDK:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Church Service Live Stream (WebRTC)</title>
  <script src="https://unpkg.com/amazon-kinesis-video-streams-webrtc/dist/kvs-webrtc.min.js"></script>
</head>
<body>
  <video id="church_stream" autoplay controls></video>
  <div id="viewer-count">Viewers: 0</div>
  <script>
    const viewer = new KVSWebRTC.Viewer({
      region: 'us-east-1',
      signalingChannelName: 'ChurchServiceWebRTC',
      accessKeyId: '<your-access-key>',
      secretAccessKey: '<your-secret-key>',
      videoElement: document.getElementById('church_stream')
    });
    viewer.start();
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

   - Use temporary AWS STS credentials for security.

3. **Host the Pages**:
   - Upload to S3 with static website hosting or integrate with your website.

## Step 8: Archive to S3
1. **Create Lambda Function**:
   - Go to **Lambda** > **Create function**.
   - **Function name**: `ChurchServiceArchiveToSS3`.
   - **Runtime**: Python 3.9.
   - **Code**:

```python
import json
import boto3
import urllib.parse
import datetime

s3_client = boto3.client('s3')
kinesisvideo_client = boto3.client('kinesisvideo')
archived_streams_client = boto3.client('kinesis-video-archived-media')

def lambda_handler(event, context):
    stream_name = 'ChurchServiceStream'
    bucket_name = 'church-service-vod-<your-account-id>'
    folder = 'recordings/'
    now = datetime.datetime.utcnow()
    start = now - datetime.timedelta(hours=3.5)
    
    response = archived_streams_client.list_fragments(
        StreamName=stream_name,
        FragmentSelector={
            'FragmentSelectorType': 'SERVER_TIMESTAMP',
            'TimestampRange': {
                'StartTimestamp': start.strftime('%Y-%m-%dT%H:%M:%S'),
                'EndTimestamp': now.strftime('%Y-%m-%dT%H:%M:%S')
            }
        }
    )
    
    for fragment in response['Fragments']:
        fragment_number = fragment['FragmentNumber']
        media = archived_streams_client.get_media_for_fragment_list(
            StreamName=stream_name,
            Fragments=[fragment_number]
        )
        s3_client.put_object(
            Bucket=bucket_name,
            Key=f"{folder}{fragment_number}.mkv",
            Body=media['Payload'].read()
        )
    
    return {
        'statusCode': 200,
        'body': json.dumps('Fragments archived to S3')
    }
```

   - Replace `<your-account-id>`.
   - Attach IAM role with `AmazonKinesisVideoStreamsReadOnlyAccess` and `AmazonS3FullAccess`.
   - Trigger via CloudWatch Events (e.g., every Sunday at 1 PM).

2. **Access Recordings**:
   - Recordings appear in S3. Transcode to MP4 using MediaConvert if needed.

## Step 9: Start and Test the Stream
1. **Start Streaming**:
   - In OBS, switch scenes and click **Start Streaming**.
   - Verify streams on the website (HLS/WebRTC), YouTube, Facebook, and Rumble.
2. **Test Playback**:
   - Check scene transitions, audio, latency (HLS: 5–10s, WebRTC: <2s), and viewer count updates.
   - Request HLS session limit increase (>10 viewers) via AWS Support [].
3. **Monitor**:
   - Use Kinesis console for ingest metrics, CloudWatch for viewer metrics, and API Gateway logs.

## Step 10: Stop Resources
1. **Stop Streaming**:
   - Stop OBS streaming.
   - Set Kinesis retention to 24 hours.
2. **Clean Up**:
   - Delete old S3 objects and unused streams.

## Cost Estimate (1,000–4,000 Viewers, 14 Hours/Month)
- **Ingest**: 4.175 GB/hour × 14 hours × $0.0085 = **$0.497/month**.
- **Consumption**: 1% of viewer data × $0.0085/GB:
  - 1,000 viewers: 18.452 GB/viewer × 1,000 × 0.01 = 184.52 GB × $0.0085 = **$1.568/month**.
  - 4,000 viewers: 18.452 GB/viewer × 4,000 × 0.01 = 738.08 GB × $0.0085 = **$6.274/month**.
- **CloudFront**: Total viewer data × $0.085/GB:
  - 1,000 viewers: 18,452 GB × $0.085 = **$1,568.42/month**.
  - 4,000 viewers: 73,808 GB × $0.085 = **$6,273.68/month**.
- **S3**: 58.45 GB × $0.023 = **$1.344/month**.
- **Restream**: ~$20/month (Basic plan for YouTube, Facebook, Rumble).
- **Lambda**: 1M requests/month (free tier, negligible).
- **API Gateway**: 1M requests/month × $3.50/M = **$3.50/month**.
- **Total**:
  - 1,000 viewers: $0.497 + $1.568 + $1,568.42 + $1.344 + $20 + $3.50 = **$1,595.33/month** (HLS) or **$1,594.80/month** (Kinesis storage).
  - 4,000 viewers: $0.497 + $6.274 + $6,273.68 + $1.344 + $20 + $3.50 = **$6,305.29/month** (HLS) or **$6,304.76/month** (Kinesis storage).

## Resources
- Kinesis Video Streams: https://docs.aws.amazon.com/kinesisvideostreams/
- WebRTC SDK: https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-js
- Restream: https://restream.io/
- Rumble Live: https://help.rumble.com/en/articles/1561686-go-live
- CloudWatch Metrics: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/
