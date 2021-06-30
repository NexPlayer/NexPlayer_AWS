# NexPlayer with AWS Elemental Integration Guide using Verimatrix DRM

## Overview
In order to use NexPlayer’s Digital Rights Management capabilities you will need to have a video player stack which also supports DRM. In this guide, we will build a VOD and Live Stream stack using a few different services, including a subset of Amazon Web Services and Verimatrix DRM. 

## VOD DRM
We will be implementing this modified VOD static flow with DRM encryption.
                
- S3 -> MediaConvert -> S3 -> CloudFront -> NexPlayer

The key components are:

1. A DRM provider. We will use **Verimatrix DRM** in this guide.
2. File hosting support. We will use AWS **S3** in this guide as it is required by other AWS Elemental tools.
3. Video encoding support. We will use AWS Elemental **Media Convert** in this guide.
4. Video content delivery support. We will use AWS **CloudFront** in this guide.
5. A capable device. Currently NexPlayer DRM is supported on **iOS, Android, PlayStation, Xbox, WebOS, Tizen** and more!

### Encryption

In order to use DRM for VOD, we’ll set up a VOD pipeline similar to the pipeline created in the VOD Static guide. If you haven’t followed that guide, we recommend you start there.

In this section you will use MediaConvert to encode a video from S3 where the output video will use DRM. Then, your client video player can request to play back the file after you provide the parameters required to authenticate the stream.

1. Go to S3
    a. Navigate to your bucket which contains your outputs folder
    b. Open outputs
    - Create a new verimatrix folder
2. Go into MediaConvert
a. Create a new job
b. Add an input
    - Input file URL: Select the video file within your S3 bucket
    
    c. Add an output group
    - DASH ISO
    - Destination
        1. S3 Bucket: Select your S3 Bucket 
        2. Location: outputs/verimatrix/
        3. Note that it will overwrite a file with the same name if it exists in this destination.
    - Toggle on DRM encryption
        1. Specify your Resource ID
            a. This will be the same ID you use for Verimatrix later
            b. I’ll use “IntegrationGuideSample” as the ResourceID
        2. Specify your System ID
            a. Verimatrix recommends using these two IDs for their Multiplay DRM on device:
            - Widevine Content Protection for MPEG DASH.
                1. edef8ba9-79d6-4ace-a3c8-27dcd51d21ed
            - Microsoft PlayReady
                1. 9a04f079-9840-4286-ab92-e65be0885f95
                
            b. See more IDs and their explanations [here](https://s3bubble.com/drm-provider-system-ids/)
        3. Specify your key provider URL. You will receive this URL from Verimatrix
            
    - Specify the bitrate of your output: 640000
        1. This is simply the default bitrate of our video. Check the bitrate of your video.
        
    d. Now go to Job Settings -> AWS integration and specify your service role
    - Use an existing role or create a new role if necessary. It needs to have access to your S3 buckets.

    e. Now click Create to start the job.
1. Navigate to CloudFront
a. Origin Domain Name: Select your S3 Bucket
b. Origin Path: /outputs/verimatrix
c. Price Class: Use Only U.S., Canada, and Europe
d. Comment: Integration Guide Verimatrix DRM
e. Create distribution
f. Now your playback URL will have the format
    - https://{cloudfronturl}/{videoname}

### Playback

Now that you have an encrypted video, you can stream your new video on your target devices. Jump to the Your Project section to set up playback.

## Live Stream DRM

### Overview

In order to use DRM for Live Streams, we’ll set up a Live Stream pipeline similar to the pipeline created in the VOD Static guide. If you haven’t followed that guide or would like to build a more complex setup, such as streaming from an encoder like OBS, you can follow our Live Stream guide. The primary difference here is that we will enable encryption within the **MediaPackage** step.

We will be using a simulated live stream sourced from S3.
- S3 -> MediaLive -> MediaPackage -> Cloudfront -> NexPlayer

The key components are:

1. A DRM provider. We will use **Verimatrix DRM** in this guide.
2. Simulated live stream support. We will use a video file hosted in AWS **S3**.
3. Live content conversion support. We will use AWS Elemental **MediaLive** in this guide.
4. Distribution creation support. We will use AWS Elemental **MediaPackage** in this guide.
5. Video content delivery support. We will use AWS **CloudFront** in this guide.
6. A capable device. Currently NexPlayer DRM is supported on **iOS** and **Android**.

### Stream from S3

You can skip this step if you already created this bucket in the other LiveStreams guide.

MediaLive has an option to take in an MP4 from the public internet. So we will need to make a public bucket to host the MP4.

1. Start by navigating to S3 in your AWS console. To keep from conflicting with other applications, I recommend you create a new bucket for this guide.
    - Your bucket will need to have a unique name.
    -  Your bucket will need to be public
        - Uncheck “Block all public access”
2. Upload one of your videos to the inputs folder within your new S3 bucket. 
    - You are able to specify for the video to loop later in this process, so you don’t need to worry about the video length. 1 minute or longer should be fine.
    - Once the video is uploaded, navigate to it and make it public.
        - Click the checkbox next to the video
        - Click “Actions” -> Make Public
    - Now click on the object and retrieve its object url.

### MediaLive MP4 Input

At this point, we encourage you to skip this section and go to the Media Package setup. We’ve left the guide in this order for clarity on the flow of data between systems.

Now, Let’s get started with [AWS Elemental MediaLive](https://console.aws.amazon.com/medialive/home).

1. Navigate to your AWS Elemental MediaLive console and click Create channel.
    - Give the channel a name.
        - NexPlayerIntegrationGuideS3DRMChannel
    - Specify the IAM Role
        - If no such role exists, create the role from a template and then use that role.
    - Specify the Channel class
        - SINGLE_PIPELINE
    - Under Input Attachments click Add
        - Click “Create input”
            1. Input name: NexPlayerIntegrationGuideLiveInputDRMMP4
            2. Input type: MP4
            3. Input Class: Use a SINGLE_INPUT
            4. URL: The object URL from your video in S3
    - Attach Input.
        1. Select the input you just created.
        2. Confirm.
    - After you’ve created and attached the input, set it to loop so that you always have content.
        - Source End Behavior: Loop
    - Create an Output
        - Click Add next to Output groups
        - Chose MediaPackage as the output group
            1. (If you have not already done so) Go to the MediaPackage setting in this guide to create a channel.
            2. Select the MediaPackage channel ID you created within MediaPackage.
            3. Confirm
        - Got to your Media Package group outputs. 
            1. Specify width and height as 1920 and 1080.
            2. Specify Codec settings: H264
                - Aspect Ratio: Specified
                - Framerate: Specified
                    - Numerator 60
                    - Denominator 1
    - Create Channel
    - Start Channel

### MediaPackage

You need to create a channel in [AWS Elemental MediaPackage](https://console.aws.amazon.com/mediapackage/home). Additionally, you will use an automated feature to set up a CloudFront distribution for that channel.

1. Under Live content, click “Create a new channel”
    - Give it an id.
        - NexPlayerIntegrationGuideLiveStreamDRMMP4Channel
    - Give it a description as well.
        - Live stream with DRM from MP4
    - To enable CloudFront click the radio button to create a CloudFront distribution for this channel.
    - At this point you can now proceed to the MediaLive setup.
        - Create a channel and specify a new output group in your MediaLive channel for the MediaPackage channel you create.
2. Now you need to create an endpoint. 
    - Click Add endpoints.
    - ID: NexPlayerIntegrationGuideS3MpegDashDRMEndpoint
    - In this example we will create an Mpeg Dash endpoint, (but you should know that Media Tailor also works with HLS).
        - Packager settings
            1. Type: DASH-ISO
    - Package encryption
        - Toggle “Encrypt content” to enabled
        - Specify your Resource ID
            1. This will be the same ID you use for Verimatrix later
            2. I’ll use “IntegrationGuideSampleLive” as the ResourceID
        - Specify your System ID
            1. Verimatrix recommends using these two IDs for their Multiplay DRM on device:
                - Widevine Content Protection for MPEG DASH.
                    - edef8ba9-79d6-4ace-a3c8-27dcd51d21ed
                - Microsoft PlayReady
                    - 9a04f079-9840-4286-ab92-e65be0885f95
            2. See more IDs and their explanations [here](https://s3bubble.com/drm-provider-system-ids/)
        - Specify your key provider URL
            1. You should have a URL from Verimatrix
        - Specify your role ARN
            1. This will be your same role that you created for MediaPackage while working on the live guide. You can find more information [here](https://docs.aws.amazon.com/mediapackage/latest/ug/setting-up-create-trust-rel.html) and SPEKE specific information [here](https://docs.aws.amazon.com/speke/latest/documentation/authentication.html).
        - Additional Configuration
            1. Key rotation interval: Uncheck this box as it is currently not supported by NexPlayer.
    - Save


### Playback

Now that you have an encrypted video, you can stream your new video on your target devices.
1. Set the URL to point to your CloudFront hosted file.
    - https://{cloudfronturl}/{videoname}
    - The formatting is very important.
2. Set your KeyServer URL
    - https://multidrm.vsaas.verimatrixcloud.net/widevine

