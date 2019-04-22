FFmpeg cuvid deinterlace problems
---

# <a name="about"></a>Description

There are 2 types of problems when using `adaptive` deinterlace with cuvid:

1. Sometimes, in the middle of transcoding, cuvid outputs frames with visible horizontal lines (as though `weave` deinterlace method was chosen). Here are two consecutive frames which demonstrate this problem (pay attention to the bottom line): [1](https://raw.githubusercontent.com/Svechnikov/ffmpeg-cuda-deinterlace-problems/master/screens/weave-problem-examples/193.jpg), [2](https://raw.githubusercontent.com/Svechnikov/ffmpeg-cuda-deinterlace-problems/master/screens/weave-problem-examples/194.jpg);
2. Occasionally, on scene changes, cuvid outputs a wrong frame, which should have been shown several seconds before (as if the frame was assigned some wrong PTS value). Here's a [listing of frames with such problem](https://raw.githubusercontent.com/Svechnikov/ffmpeg-cuda-deinterlace-problems/master/screens/wrong-frame-example/153.png).

This repository is intended to prove that such problems exist and to show, how to solve them.

# <a name="preparing"></a>Preparing steps

In order to reproduce the problems, you should have:

1. Linux environment;
2. Nvidia graphic card with cuvid/nvenc support and latest drivers installed (I tested on GTX 1080, GTX 1050 and 418.56 drivers);
3. Docker installed (for building and running FFmpeg). Using docker is not mandatory, but recommended, as it simplifies the process of building;
4. [Nvidia docker runtime](https://github.com/NVIDIA/nvidia-docker/) installed (if using docker).

Building the image:

`docker build images -f images/Dockerfile -t ffmpeg-cuvid-test`

# <a name="running"></a>Reproducing the weave deinterlace problem

I prepared two samples with which the problems are reproduced.

For the first problem you should use the `weave-problem.ts` sample:

`docker run -it -v $(pwd)/samples:/samples --rm ffmpeg-cuvid-test ffmpeg -y -hwaccel cuvid -c:v h264_cuvid -deint adaptive -copyts -i /samples/input/weave-problem.ts -map 0:0 -c:v h264_nvenc -r 50 /samples/output/weave-problem.ts`

After the ffmpeg process completes try playing the output file `samples/output/weave-problem.ts`.
Approximately on the 5-th second you will see, that the horizontal lines appear (it's especially noticable on the bottom moving text).
After a while the lines disappear and reappear a couple of times.

You can examine each frame as a jpg image:

`docker run -it -v $(pwd)/samples:/samples --rm ffmpeg-cuvid-test sh -c 'mkdir -p /samples/output/weave-problem-frames && rm -rf /samples/output/weave-problem-frames/* && ffmpeg -i /samples/output/weave-problem.ts /samples/output/weave-problem-frames/%03d.jpg'`

You will see, that the first moment, when the lines appear, is the 194-th frame.

# <a name="running"></a>Reproducing the wrong frame problem

You should use the `wrong-frame.ts` sample:

`docker run -it -v $(pwd)/samples:/samples --rm ffmpeg-cuvid-test ffmpeg -y -hwaccel cuvid -c:v h264_cuvid -deint adaptive -copyts -i /samples/input/wrong-frame.ts -map 0:0 -c:v h264_nvenc -r 50 /samples/output/wrong-frame.ts`

In the output `samples/output/wrong-frame.ts` on the 4-th second you will notice a flickering frame - this is our problem.

You can examine each frame as a jpg image:

`docker run -it -v $(pwd)/samples:/samples --rm ffmpeg-cuvid-test sh -c 'mkdir -p /samples/output/wrong-frame-frames && rm -rf /samples/output/wrong-frame-frames/* && ffmpeg -i /samples/output/wrong-frame.ts /samples/output/wrong-frame-frames/%03d.jpg'`

You will see, that the 153-rd frame is out of the sequence.

# <a name="running"></a>Fixing the problems

The reason behind the problems is that sometimes `CUVIDPARSERDISPINFO` has property [progressive_frame](https://github.com/FFmpeg/FFmpeg/blob/master/libavcodec/cuviddec.c#L513) equal to 1 (which is wrong, the videos are interlaced).
In order to fix the problem we should check if the video is interlaced or progressive in the beginning of a video sequence (the best place for that is [cuvid_handle_video_sequence](https://github.com/FFmpeg/FFmpeg/blob/master/libavcodec/cuviddec.c#L107) function).
And then we just use this information instead of the faulty `progressive_frame` in `CUVIDPARSERDISPINFO`.

I prepared a patch [images/fix.patch](https://github.com/Svechnikov/ffmpeg-cuda-deinterlace-problems/blob/master/images/fix.patch). You can test the patch using a special docker-image:

`docker build images -f images/Dockerfile.fixed -t ffmpeg-cuvid-test`

If you run the commands again, you will see neither the weavy output, nor the wrong frames.

I tested the patch on multiple video streams (both progressive and interlaced), haven't found any problems.
