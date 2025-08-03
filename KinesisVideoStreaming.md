# AWS Kinesis Video Streams Multi-Camera Streaming Guide

This guide updates the setup for streaming a church service using Amazon Kinesis Video Streams, CloudFront, and S3, with a multi-camera setup (three cameras: pulpit, choir, congregation) managed via OBS Studio, WebRTC for low-latency viewers, and Restream for YouTube/Facebook Live.

## Prerequisites
- **AWS Account**: Sign up at https://aws.amazon.com.
- **IAM Permissions**: Create an IAM user/role with permissions for Kinesis Video Streams, CloudFront, S3, and Lambda (use policies like `AmazonKinesisVideoStreamsFullAccess`, `CloudFrontFullAccess`, `AmazonS3FullAccess`, `AWSLambdaFullAccess`).
- **Hardware/Software**:
  - Three 1080p cameras (e.g., DSLR, webcam, or IP cameras).
  - Microphones and audio mixer.
  - Computer with OBS Studio (https://obsproject.com/).
  - Stable internet (5–10 Mbps upload).
- **Church Website**: For embedding HLS/WebRTC streams.
- **Restream Account**: Sign up at https://restream.io/ for YouTube/Facebook streaming. Obtain RTMP ingest URL and key.
- **Social Media Accounts**: YouTube and Facebook accounts with live streaming enabled.

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
1. **Install OBS Studio**:
   - Download from https://obsproject.com/.
2. **Add Camera Sources**:
   - Create three scenes (“Pulpit”, “Choir”, “Congregation”):
     - Add **Video Capture Device** for each camera (via HDMI/USB) or **Media Source** for IP cameras (e.g., `rtsp://<camera-ip>/live`).
   - Add **Audio Input Capture** for the microphone/mixer.
3. **Configure Scene Switching**:
   - Use OBS’s **Scene Collection** or **Auto Scene Switcher** plugin for transitions.
4. **Configure Output for Kinesis**:
   - Go to **Settings** > **Stream**:
     - Service: Custom.
     - Server: `rtmp://<ingest-endpoint>.kinesisvideo.us-east-1.amazonaws.com:1935/app`.
     - Stream Key: `ChurchServiceStream`.
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
   - Add YouTube and Facebook as destinations in Restream Studio.
   - Copy the Restream RTMP URL and stream key (e.g., `rtmp://live.restream.io/live/<stream-key>`).
2. **Configure OBS for Restream**:
   - In OBS, go to **Settings** > **Stream**:
     - Add a second streaming service (requires OBS multi-RTMP plugin: https://github.com/sorayuki/obs-multi-rtmp).
     - Service: Custom.
     - Server: Restream RTMP URL.
     - Stream Key: Restream stream key.
   - Start streaming to both Kinesis and Restream simultaneously.

## Step 5: Configure Amazon CloudFront for HLS Delivery
1. **Navigate to CloudFront**:
   - Go to **CloudFront**.
2. **Create a Distribution**:
   - **Origin**: Kinesis HLS URL (e.g., `https://<endpoint>.kinesisvideo.us-east-1.amazonaws.com/hls/v1/getHLSMasterPlaylist.m3u8?StreamName=ChurchServiceStream`).
   - **Protocol**: HTTPS only.
   - **Cache behavior**: `Managed-CachingOptimized`.
   - **Price class**: All edge locations.
   - Click **Create distribution**. Note the **Distribution Domain Name**.

## Step 6: Embed Streams on the Church Website
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
  <script src="https://vjs.zencdn.net/7.20.3/video.min.js"></script>
  <script>
    var player = videojs('church_stream');
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
  <script>
    const viewer = new KVSWebRTC.Viewer({
      region: 'us-east-1',
      signalingChannelName: 'ChurchServiceWebRTC',
      accessKeyId: '<your-access-key>',
      secretAccessKey: '<your-secret-key>',
      videoElement: document.getElementById('church_stream')
    });
    viewer.start();
  </script>
</body>
</html>
```

   - Use temporary AWS STS credentials for security (not hardcoded keys).

3. **Host the Pages**:
   - Upload to S3 with static website hosting or integrate with your website.

## Step 7: Archive to S3
1. **Create Lambda Function**:
   - Go to **Lambda** > **Create function**.
   - **Function name**: `ChurchServiceArchiveToS3`.
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
    start = now - datetime.timedelta(hours=1)
    
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

## Step 8: Start and Test the Stream
1. **Start Streaming**:
   - In OBS, switch scenes and click **Start Streaming**.
   - Verify streams on the website (HLS/WebRTC), YouTube, and Facebook Live.
2. **Test Playback**:
   - Check scene transitions, audio, and latency (HLS: 5–10s, WebRTC: <2s).
   - Request HLS session limit increase (>10 viewers) via AWS Support.
3. **Monitor**:
   - Use Kinesis console for ingest metrics and CloudWatch for CloudFront metrics.

## Step 9: Stop Resources
1. **Stop Streaming**:
   - Stop OBS streaming.
   - Set Kinesis retention to 24 hours.
2. **Clean Up**:
   - Delete old S3 objects and unused streams.

## Cost Estimate (1,000–4,000 Viewers)
- **Ingest**: 4.175 GB/hour × 4 hours × $0.0085 = $0.142/month.
- **Consumption**: 1,000 viewers = 52.72 GB × $0.0085 = $0.448/month; 4,000 viewers = 210.88 GB × $0.0085 = $1.792/month.
- **CloudFront**: 1,000 viewers = 5,272 GB × $0.085 = $448.12/month; 4,000 viewers = 21,088 GB × $0.085 = $1,792.48/month.
- **S3**: 16.7 GB × $0.023 = $0.384/month.
- **Restream**: ~$20/month (Basic plan for 2 platforms).
- **Total**:
  - 1,000 viewers: $0.142 + $0.448 + $448.12 + $0.384 + $20 = **$469.09/month** (HLS) or **$468.85/month** (Kinesis storage).
  - 4,000 viewers: $0.142 + $1.792 + $1,792.48 + $0.384 + $20 = **$1,814.80/month** (HLS) or **$1,814.56/month** (Kinesis storage).
- **Note**: Multi-camera doesn’t increase ingest bitrate. Restream adds ~$20/month. WebRTC reduces consumption costs but requires custom scaling.

## Resources
- Kinesis Video Streams: https://docs.aws.amazon.com/kinesisvideostreams/
- WebRTC SDK: https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-js
- Restream: https://restream.io/
- OBS Multi-RTMP Plugin: https://github.com/sorayuki/obs-multi-rtmp
