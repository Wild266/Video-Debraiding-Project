# Akshan Sameullah

## Debraiding Video into Motion Vectors and Code Features

In my COMP590: Video Compression class taught by Professor Ketan Mayer Patel, we dive into the concept of Video Compression, a technique that enables large videos to be represented in a variety of ways that take advantage of common video patterns such as spatial coherence (the idea that pixels in videos tend to be similar to surrounding pixels) and temporal coherence (video pixels tend to be similar to other pixels in those areas in adjacent time steps) to reduce the load of transmitting video files across the internet for tasks such as streaming. One such standard for representing videos is known as H.264. The goal of this project is to design and implement a small research-and-engineering project that “debraids” (decomposes) video into motion vectors and related codec features using existing tools. This will allow for comparison of codec motion vectors, extraction of features from frames, and exploring how encoder settings affect different codec features. This is also incredibly beneficial for my understanding of how videos are compressed in the H.264 format. The learning objectives were to gain an understanding of motion estimation in codecs and what debraiding video really entails and how motion vectors, QP, and residuals interact. Ultimately, I learned useful engineering skills for video and how motion/spatial complexity can inform streaming decisions by gaining hands-on experience with a novel and interesting project.

Essentially, what I am doing here is implementing existing tools to peel open a video and show the raw parts of it according to how they are defined in the H. 264 standard. If we think about what a video is, it’s essentially a bunch of frames shown in rapid succession. H.264 at a high level, compresses this sequence by portioning these frames into groups called blocks using schemas according to the standard. Now, it compares how pixels moved in adjacent blocks to get to subsequent blocks. This data describes a codec known as motion vectors, as discussed in more detail below. This isn’t usually enough to get us to the next block, so h.264 standardizes a way to adjust the predicted block we get from current blocks such that it exactly matches future blocks using what are known as the residuals. We then shrink these values by throwing away subtle details according to quantization parameters, resulting in a blurrier video that involves more loss depending on how aggressively we compress. There is a layer above this, known as the Network Abstraction Layer, which aggregates this information into units and assigns each NAL unit a type as specified in the H.264 standard. 

### The tools that I use in this project include:

- ffmpeg / ffprobe  
  A well known tool used to decode raw video up to the frame level. Open source, yet an industry standard tool  
- mv-extractor  
  An open source tool that uses ffmpeg and ffprobe to extract the motion vectors from the source video and overlay a them in realtime.  
- ffmpeg-debug-qp  
  An open source tool that uses ffmpeg to show us macroblock QP values.  
- h264_analyze (h264bitstream)  
  An open source tool that prints NAL header fields and slice info from the 264 bit stream.  
- JM 16.1 with XML trace  
  A well known decoder that extracts all remaining elements not pulled in above tools into an XML format.  

---

### Commands 1: Converting MP4 to raw 264 bit stream

```bash
ffmpeg -hide_banner -y -i input.mp4 -c:v copy -bsf:v h264_mp4toannexb -f h264 input.264
```

The above command converts our existing video, input.mp4, into a raw h264 bit stream file. This is helpful for the ingestion of the video into the subsequent tools (specifically h264_analyze and JM, which require the Annex B format) that we use to debraid it into specific codecs. The MP4 file contains data that we don’t actually need to look at either such as audio. This takes out specifically the video and puts it into the Annex B format using h264_mp4toannexb. The flag -c:v copy ensures that we don’t recompress the video and save the raw guts of it.

*Figure 1. The first 5 lines of input.264*

The resulting file is a giant blob of mostly non-decipherable bytes undisplayable in github (to humans anyway). These bytes include the start codes of the video followed by representations of other parameters representing each NAL unit as defined in the h264 standard. For example, the second byte after the start code, defines the nal_unit_type, which could be one of these values: 5 = IDR slice, 7 = SPS, 8 = PPS. The bytes following the NAL parameters are the entropy encoded bits, appearing like random characters when displayed in Figure 1.

---

### Commands 2: Motion vector extraction

**C1 Raw to a json:**

```bash
ffprobe -hide_banner -v error  -export_side_data +mvs  -select_streams v:0  -show_frames  -show_entries frame=pkt_pts_time,pict_type,side_data_list  -of json input.mp4 > mv_frames.json
ffprobe -hide_banner -v error -export_side_data +mvs -select_streams v:0 -show_frames -of flat "input.mp4"
```

**C2 Overlay:**

```bash
ffplay -flags2 +export_mvs -vf "codecview=mv=pf+bf+bb" input.mp4
```

**C3 mv-extractor tool use:**

```bash
docker run --rm -it -v "${PWD}:/home/video_cap" lubo1994/mv-extractor python3.12 extract_mvs.py vid_h264.mp4 --verbose
```

**C4 mpegflow tool use:**

```bash
nmake /A clean
nmake mpegflow.exe FFMPEG_DIR=C:\vcpkg\installed\x64-windows
.\mpegflow.exe --raw "examples\mpi_sintel_final_alley_1.avi" 1>out.txt 2>err.txt
```

To extract the motion vectors, an extraction implementation exists with ffmpeg, as shown in Command 2 C1 where we use the flag -export_side_data +mvs to extract the motion vectors as we go through the video, and the flag -vf codecview=mv=pf+bf+bb in C2 highlights explicitly the parts of the motion vectors we are extracting. Motion vectors are essentially the vector from a sixteen by sixteen pixel macroblock to the most similar sixteen by sixteen macroblock in a different frame. Pf represents the forward motion vector to the next p-frame, bf is forward to the next B-frame, and bb is to the previous B-frame. We can overlay this over our video by replacing ffmpeg with ffplay C1→ C2 to see the motion vectors on the video as shown in Figure 2. I noticed they were small when displaying them and so I downloaded a smaller resolution version of the same input.mp4 and ran C2 again to get Figure 3 with bigger motion vector arrows. This makes sense because the macroblock size was held constant (although h264 can accommodate other block sizes as well), and so there are fewer motion vectors and fewer/bigger arrows. Not pictured due to my hardware constraints (laptop runs out of RAM when running this docker container), but I also used other tools such as mv-extractor to reproduce these values (C3). C1 the ffprobe commands gave me fairly high level data about the motion vectors (shown in Figure 4) So I used C4 to run another tool called Mpegflow and extract the raw motion vector values (Figure 5) which give me forward mv.dst_x, mv.dst_y, mvdx, mvdy explicitly.

*Figure 2. Motion vectors overlaid over a video of birds flying next to a sunset*

*Figure 3. Motion vectors overlaid over a smaller resolution video version. Note that the motion vector arrows appear bigger and more spaced out.*

*Figure 4. Metadata obtained about motion vectors using C1*

*Figure 5. Raw Motion Vector values obtained using mpegflow (C4). Values shown are specifically for mv.dst_x, mv.dst_y, mvdx, mvdy for the specified frame.*

---

### Commands 3: Quantization Parameter

```bash
.\ffmpeg_debug_qp.exe ../input.mp4 2> qp_debug.log
```

When we run compression, we throw away some of the video’s detail and we control how much this quality reduction is using the quantization parameter. Essentially, the bigger this parameter is the more detail we lose in a macroblock.  
When we run this command, we get the list of quantization parameters written to qp_degub.log as shown in Figure 6. This gives us the two digit quantization parameters for each frame (the tool did not separate them with whitespace, I am continuing to investigate why). Each row of this file gives us the quantization parameters for a row of macroblocks in the frame. In areas of lower quantization parameters, we are spending more bits to get higher quality values while in areas of higher quantization parameter values we spend less and get more compressed macroblocks.

*Figure 6. QP values obtained using Command 3.*

---

### Commands 4: Network Access Layer headers and slice metadata

```bash
h264_analyze.exe input.264 > nal_headers.txt
```

I found a tool that allows us to get data on each NAL header and their payload in a much more easy to digest format using our raw bitstream. Using Commands 4, we can parse the raw .264 bitstream using a tool called h264_analyze to search for and identify NAL headers and their associated types and parameters. Each section marked “!!” notes a new NAL header of type nal_unit_type. For example, in Figure 7 we see an NAL of type 7 which is the SPS NAL unit and the parameters give us constants useful for identifying the color, resolution, etc.

*Figure 7. NAL headers found using h264_analyze*

---

### Commands 5: Expansive toolset using JM

```bash
jm_16.1_xmltrace_v1.5\jm_16.1\bin\ldecod.exe -s -i input.264 -o decoded.yuv -xmltrace trace.xml

ffmpeg -f rawvideo -pixel_format yuv420p -video_size 1920x1080 -i decoded.yuv -c:v libx264 decoded.mp4
```

Tolerant version:

```bash
ffmpeg -i input.mp4 -c:v libx264 -preset veryslow -crf 18 -profile:v high -level:v 4.0 -x264-params ref=4:dpb_size=4:b-pyramid=0 -an -f h264 jm_clean.264

jm_16.1_xmltrace_v1.5\jm_16.1\bin\ldecod.exe  -s -i jm_clean.264 -o decoded.yuv -xmltrace trace.xml

ffmpeg -f rawvideo -pixel_format yuv420p -video_size 1920x1080 -i decoded.yuv -c:v libx264 decoded.mp4
```

I discovered after extracting the components we found thus far that there is a tool known as JM which is the industry-trusted tool for h.264. Using the decoder executable I compiled after removing certain warning and error catches, we can get a decoded YUV file which is the video in YUV420 pixels. The command also gives us an XML file with relevant information to the decoded values displayed in a relatively pretty way. Figure 8 shows this for a macroblock labeled and displaying the Quantization Parameters for a macroblock. We can use ffmpeg to re-encode the decoded YUV file and reconstruct the original input video we started with. Note, JM is a little sensitive and could not completely decode the full video (which was randomly found on the open source web). This may be due to the warnings I removed in the source code to run JM in the first place. For the repository’s specific video file I also included some tolerant commands to demonstrate the video is decodable and recoverable for testing purposes.

*Figure 8. A macroblock’s quantization parameters discovered using JM in the trace XML file*
