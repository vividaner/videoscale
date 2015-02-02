# videoscale

[Problem]
Use SIMD vector operations to accelerate the upscaling of 720p to 1080p video.

Provide an implementation that uses SSE or ARM NEON intrinsics or assembly. Assume any 2d separable filter for upscaling.

Deliver source code (with comments) that is optimized for memory access by considering cache utilization, eliminating unaligned data accesses and avoiding register spills.


[Implementation]
1) Straight forward bilinear interpolation (assuming scaling from (w,h) to (x,y))
* Two pass filter: (w,h) -> (x,h) -> (x,y)
* Unified filter function (specify stride) for code reusing
* Define scale/offset to control accuracy of integer division ( a/b -> ((a<<scale)/b+offset)<<scale )
* Detect the corner case when w==x and/or h==y

2) Implement the scaling as three modes
a. Normal mode: 
upsample the picture using the bilinear interpolation with two pass filters

b. Precache mode:  
Improve the bilinear interpolation with pre-computed weights. 
* Identical locations of source pixels pairs for each line, also
* Identical weights for each line. Compute them once and use for the rest.
* Pre-scale weights as well: weightTable[i] = ( ( (i * srcLen) % dstLen ) << scale ) / dstLen

c. SIMD + Precache mode:
Based on mode b, use the SSE to accelerate the interpolation.
* giving scale=8, intermediate result requires < 2^8*2^8+2^7 < 2^32
* loading 4 source pixel pairs and 4 weight pairs simultaneously
* emulate _mm_mul_epi32 with _mm_mul_epu32 and shifting/shuffling
* align memory for weight tables, they can be sequentially loaded
* source memory is still randomly accessed (undertiminstic memory location for source pixel pairs)

3) Test
I tested the video upsampling from 720p to 1080p in my macbook and recorded the complexity result which is as follows:
*Normal mode: about 63.60ms/frame
calculate weights cost 20% time, calculating results cost 72% time

*Precache mode: 30.80ms/frame
60% time for calculating results, 40% for memory access

*SIMD mode: 27.10ms/frame
45% time used for loading source, 20% time used for storing to destination

[usage]

scale [-i input=/dev/stdin] [-o output=/dev/stdout] [-w width=176] [-h height=144] [-x scaledWidth=352] [-y scaledHeight=288] [-f framenum=1] [-a accuracy=8] [-m method=1]

The values in the usage are default values. -m controls method to be used
Method 0: plain bilinear, 1: improved bilinear, 2: SSE2 based SIMD bilinear

Example:
./scale -i city720p.yuv -o output1080p.yuv -w 1280 -h 720 -x 1920 -y 1080 -f 2 -m 2 



