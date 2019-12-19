Discord Intel GPU Encoding Issues
=================================

This repo is the clone of [https://github.com/Intel-Media-SDK/MediaSDK.git](https://github.com/Intel-Media-SDK/MediaSDK.git). The goal is to reproduce a performance issue when converting from BGRA to NV12 using video memory followed by encoding.

Intel SDK Version: Intel(R) Media SDK 2018 R2\
Graphics: Intel(R) HD Graphics 630\
Driver Version: 26.20.100.7372\
Driver Date: 10/24/2019

Integration to Discord
----------------------
Intel QuickSync GPU video encoding is integrated into Discord.  System memory is used for webcam and generally video memory is used for window/screen capture. To verify encoder used, please open audio/video statistics by clicking on Voice Connected link while in a call. Select Debug in popup and look for Encoder under Outbound pane. It should say `intel` or `intel: direct 3d` for system memory and video memory, respectively.


I-Frame Generation
------------------
WebRTC frequently updates target video bitrate. We use `MFXVideoENCODE_Reset` to update bitrate. This generates an I frame, which is undesirable.

### Setup
1. Using tutorial `simple_3_encode`
2. Re-targeted the solution for Windows SDK Version 10.0.17763.0 (Visual Studio 2017)
2. Modify encoding parameters, I use https://github.com/open-webrtc-toolkit/owt-client-native/blob/master/talk/owt/sdk/base/win/msdkvideoencoder.cc#L164-#L196
3. At every 100th frame, increase target bitrate and maximum bitrate by 10%
4. Print out encoded frame type

### Results
I use command lime arguments `-g 1920x1080 -b 6000 -f 30/1`. Observer that  I frame is inserted after every reset.

BGRA to NV12 conversion
-----------------------
 Issue: [https://github.com/Intel-Media-SDK/MediaSDK/issues/1827](https://github.com/Intel-Media-SDK/MediaSDK/issues/1827).

 Update: Since encoder can directly consume `DXGI_FORMAT_B8G8R8A8_UNORM`, we do not need to implement format conversion for video memory.

### Setup
1. Using tutorial `simple_4_vpp_resize_denoise_vmem`
2. Re-targeted the solution for Windows SDK Version 10.0.17763.0 (Visual Studio 2017)
3. VPP input FourCC is `MFX_FOURCC_RGB4`
4. Output resolution same as input resolution
5. Replace `LoadRawFrame` with `LoadRawRGBFrame`
6. Add timing around `MFXVideoSession::SyncOperation`
7. Add video encoder parameter initialization and `MFXVideoENCODE::Query()` right before VPP parameters initialization

### Results

I use 555 frames of Big Buck Bunny at 1920 x 1080 resolution. When only doing BGRA to NV12 conversion (without step 6), it takes around 1350 ms for `MFXVideoSession::SyncOperation` to execute, about 2.7 ms per frame. When initializing and validating video encoding parameters (`MFXVideoENCODE::Query`), the same sync operation takes about 7700 ms to execute, about 15.4 ms per frame. Exactly the same result is obtained by using empty frames.

When using system memory, I do not observer any performance penalty, BGRA to NV12 conversion and H264 encoding take about 2.27 ms and 0.9 ms, respectively.

