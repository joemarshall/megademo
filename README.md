# Epic Megagrants Proposal - Vulkan Video Rendering for Android

This proposal is to merge my existing work on vulkan accelerated rendering of video on Android into Unreal Engine.

# What does this fix?

On the Android platform video playback in Unreal is **broken** when using Vulkan. This becomes increasingly obvious when you are building for standalone VR platforms such as Meta Quest 3, and you try to play a 360 degree video, which are typically filmed at 6k or greater resolution. In GLES, it is possible to play videos at full frame rate with the headset running at steady game rates. Build the same code in Vulkan, and frame rate drops massively, the headset is unusable, and everything heats up. With my plugin, videos play smoothly in Vulkan and the headset no longer drops frames. 

Check out the [demo video and apk below.](#Demos)

# What is the proposed budget

The proposed budget covers an estimate of developer time, and some additional hardware costs to enable me to work more effectively on this; given the time taken for engine / plugin rebuilds it makes sense to upgrade my development hardware to reduce iteration time. I also budget for Android devices to test on.

|Item|Unit cost|Number|Total|
|----|---------|------|-----|
| Developer days  | 850 | 40 | $34000 | 
| Updated development machine  | 5000 | 1 | $5000 |
| Test hardware | 500 | 8 | $4000 |
| **Final Total** | || **$41000** |


# Why is Android video so bad in Unreal?
Video in Unreal Android is handled by Mediaplayer14.java . This plays video using Android Java and copies it to Unreal C++. Under OpenGL, it is possible to pass textures across the Java / C boundary. However the Java mediaplayer API has no support for Vulkan, so it isn't possible there. Instead in Vulkan, each video frame is copied to a CPU buffer, which is then loaded into a vulkan texture. Unsurprisingly when you start playing with typical 360 video resolution such as 6016x2560@25fps, this means transferring approx 1.5 gigabytes of data per second across the CPU / GPU boundary. This also removes the ability to make use of hardware support for colour space conversions, which again kills performance.

# How will my project fix this?
This project will add an alternative media player for Android coded in Native c++. This will enable unreal to make full use of hardware decoding and colour space conversion, and keep video textures on GPU throughout the process.

I have built a proof of concept plugin which does the following:

1. Decodes video using NDK media codec.
2. Loads the Android Hardware Buffers produced directly into Vulkan using the VK_ANDROID_external_memory_android_hardware_buffer and VK_KHR_sampler_ycbcr_conversion extensions. Because Unreal doesn't support these extensions by default I use a Vulkan layer in an external shared library to correctly modify the initialisation calls and add a VkPhysicalDeviceSamplerYcbcrConversionFeatures structure to the create info. 
3. Renders them into a media texture  through adding raw vulkan calls to the RHICommandBuffer. 

This works quite well, but (a) having to hack the vulkan initialisation using an override layer is a pain, and (b) it means video lives partially outside the existing vulkan rhi (it is using the same device and queues, but bypassing the RHITexture objects etc.) With engine support for Android hardware buffers it would also be possible to render video direct to swapchain (i.e. to use video output textures directly as inputs to a material shader), which could further improve the performance with high resolution videos, and also make it much easier to deal with HDR video where supported.

# The Plan

I propose the following multi stage plan. At each stage, Unreal users have access to a working open source solution for high performance video on Android.

## 1: Release existing plugin as open source (7 days)
My existing plugin, whilst it is a bit of a hack, works, and gives users of Unreal 5.3 upwards the possibility of using high quality video on Android. It is also a useful educational tool for Vulkan developers, as currently I could not find a single example of Android video plus audio decoding, rendering and colour conversion out there, and much of what is currently available doesn't follow vulkan specs correctly, so throws validation errors and won't work on some arm GPU devices.

In this phase of the work I will ensure the code is consistent with Unreal coding style, then release my source code under a permissive open source license on GitHub, and upload it to Unreal marketplace. 

## Phase 2: Engine support for VK_KHR_sampler_ycbcr_conversion (3 days)
This is a relatively trivial change, to add this extension to VulkanExtensions.cpp. It needs adding into the engine because there is a corresponding feature flag structure that needs adding to the create info chain when this extension is enabled.

After this phase the Unreal community has a high performance video solution which doesn't require any nasty hacks (the added vulkan layer) to use.

## Phase 3: Engine support for VK_ANDROID_external_memory_android_hardware_buffer (30 days)
This work would add engine support for VkImages created based on Android AImage native hardware buffers. This would be a really useful feature because 
A) it removes the need for the video playback plugin to create its own Vulkan command lists and pipelines. 
B) It would also create the possibility for advanced users of zero texture copy rendering of video, by rendering using shaders directly onto the swapchain buffers; this capability is likely to become increasingly important as resolutions increase and people start using stereoscopic 360 videos.
C)  Other uses of hardware buffers may also be possible, e.g. streaming video via webrtc, screen capture, 

# Why me?
1) I've done the initial work on integrating NDK media into Unreal engine already. I did some work on this as part of a contract for a developer of 360 video experiences, for whom I also developed custom webRTC streaming solutions using Unreal Engine.

2) I'm an experienced and professional developer, having programmed professionally for almost 30 years; I do freelance work primarily on open source projects, and as such the quality of my work can be seen in my contributions to various projects across GitHub, including pull requests to major projects such as the python interpreter (cpython).  

3) I am also a part time academic in the school of computer science at Nottingham university, UK. My academic work involves technical work in an immersive production studio, where we use both Unreal and Unity in conjunction with 360 capture, volumetric video, mocap and a range of experimental VR input and output technologies. As such I've got a range of experience integrating new technology into game engines and thanks to our collection of standalone vr headsets and phones I have extensive knowledge of low level optimisation of 3d rendering on Android.



# Demos

The following video is captured on Meta Quest 3, playing a 7680x3840 x 30 FPS H265 MP4 file encoded at a bitrate of 117000 kb/s. When the screen says "vulkan_vide", that is my plugin, when it says 'old_slow', that is the standard Unreal Android Media Player. The FPS displayed is the headset FPS. As you can see, when using the standard media player, the headset drops to an unusable 15 FPS, which on a VR headset is sickness inducing and tears as you move your head. With direct Vulkan video, the headset keeps a steady 72FPS, and head tracking remains smooth and lovely. 

If you have access to a Quest 3 and want to see this in action yourself, download the [demo apk (200MB)](https://github.com/joemarshall/megademo/releases/latest/download/quest_demo.zip). Apologies, it is massive, because it contains a 10 second 360 video clip....

Similarly, if you have a recent Android phone, please do test the phone demo, which displays the same 360 degree MP4 file and runs smoothly at 60FPS on my mid range phone (Oppo Find X3 Neo) as opposed to <10 fps using built in media support.


https://github.com/joemarshall/megademo/assets/1436795/bc84ce8a-07ef-4d95-a95d-85f81bc05c50


