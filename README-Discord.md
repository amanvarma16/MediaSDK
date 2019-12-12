Discord Intel GPU Encoding Issue
================================

This repo is the clone of [https://github.com/Intel-Media-SDK/MediaSDK.git](https://github.com/Intel-Media-SDK/MediaSDK.git). The goal is to reproduce a performance issue when converting from BGRA to NV12 using video memory followed by encoding.

Intel SDK Version: Intel(R) Media SDK 2018 R2
Graphics: Intel(R) HD Graphics 630
Driver Version: 26.20.100.7372
Driver Date: 10/24/2019

Setup
-----
1. Re-targeted the solution for Windows SDK Version 10.0.17763.0 (Visual Studio 2017)
2. VPP input FourCC is `MFX_FOURCC_RGB4`
3. Output resolution same as input resolution
4. Replace `LoadRawFrame()` with `LoadRawRGBFrame()`
5. Add timing around `MFXVideoSession::SyncOperation()`
6. Add video encoder parameter initialization and `MFXVideoENCODE::Query()` right before VPP parameters initialization

Results
-------
I used 555 frames of Big Buck Bunny at 1920 x 1080 resolution. When only doing BGRA to NV12 conversion (without step 6), it takes around 1350 ms for `MFXVideoSession::SyncOperation()` to execute, about 2.7 ms per frame. When initializing and validating video encoding parameters (`MFXVideoENCODE::Query()`), the same sync operation takes about 7700 ms to execute, about 15.4 ms per frame. You can reproduce the same exact result without specifying input/output files (just using 1000 black frames).

When using system memory, I do not observer any performance penalty, BGRA to NV12 conversion and H264 encoding take about 2.27 ms and 0.9 ms, respectively.

We dig into this issue with `Windows Performance Analyzer` and `GPUView`. We observed about five times lower GPU utilization when validating encoding parameters. Conversion from BGRA to NV12 still takes about 2.7 ms per frame, however, there is a very clear pause between GPU operations that seems to line up with vsync.

When specifying codec `MFX_COEC_HEVC` or `MFX_CODEC_MPEG2` instead of `MFX_CODEC_AVC`, we did not observer any pauses.


Integration to Discord
----------------------
Intel QuickSync GPU video encoding is integrated into Discord. By default, system memory is used. You can enable video memory by enabling Experimental Encoders under Voice & Video settings. You can open audio/video statistics by clicking on Connection Info/Debug while in a call. Under Outbound pane, look for Encoder should say `intel` or `intel: direct 3d` for system memory and video memory, respectively.
