# FEATURES and QUALITY METRICS DATASETS for VIDEO CODING in DASH

This repository is related to the paper "Features and Quality Metrics Datasets for Video Coding in DASH" submitted to the journal "Scientific Data" and comprises sample Bash source code employed to extract features and compute VQA metrics from 4k videos, with the objective of creating datasets that are publicly available on figshare. Besides, we also provide sample Python source code for the combination and processing of the datasets. And finally, we present web pages displaying figures and statistical analyses conducted with all the datasets.

# Table of contents
1. [Sample comands used to extract features and compute VQA metrics](#sample-comands-used-to-extract-features-and-compute-vqa-metrics)
2. [Dataset handling](#dataset-handling)
3. [Dataset statistical analysis and plots](#dataset-statistical-analysis-and-plots)

<!--## Cite
This work has been published in xxx:

If you use part of or all the datasets, please cite that paper.
-->

## Sample comands used to extract features and compute VQA metrics

In the following we assume that you have the ```wget```, ```ffmpeg``` and ```siti``` tools installed. Here are the steps you have to follow with the aim of extracting features and computing VQA metrics for one 4k video (to illustrate each of these steps, we will use the video file ```Vlog_2160P-19f9.mkv``` from the ```YouTube User Generated Content (UGC) dataset```).

### Download sample video

First we obtain the file from the UGC dataset:

```bash
wget https://storage.googleapis.com/ugc-dataset/original_videos/Vlog/2160P/Vlog_2160P-19f9.mkv 
```

### Video standardisation

Taking into account the diverse range of containers, chroma sampling, and bit depths of the videos downloaded from databases referenced in paper, the next step involve standardizing these characteristics:

```bash
ffmpeg -hide_banner -i "Vlog_2160P-19f9.mkv " -an -c:v libx264 -crf 0 -preset ultrafast -pix_fmt yuv420p "Vlog_2160P-19f9_3840x2160_crf0.mp4"
```

### Video temporal splitting and spatial downscaling

After the standardization process, each video is divided into 2-second chunks with vertical resolutions of 240, 360, 480, 720, and 1080 pixels:

```bash
ffmpeg -hide_banner -i "Vlog_2160P-19f9_3840x2160_crf0.mp4" -filter_complex "[0:v]yadif,split=7[out1][out2][out3][out4][out5]" -map "[out1]" -s 1920x1080 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_1920x1080_chunk%4d_crf0.mp4" -map "[out2]" -s 1280x720 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_1280x720_chunk%4d_crf0.mp4" -map "[out3]" -s 854x480 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_854x480_chunk%4d_crf0.mp4" -map "[out4]" -s 640x360 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_640x360_chunk%4d_crf0.mp4" -map "[out5]" -s 426x240 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_426x240_chunk%4d_crf0.mp4"
```

### Features extraction

By utilizing the ffprobe tool with the signalstats and entropy filters, as well as employing the Python siti package v1.4.5 in accordance with 2021 ITU-T Recommendation P.910, we extract a collection of features from all the chunks created in the previous step and save them into json files. Here we show the commands for the first chunk of the video in 240 resolution:

```bash
ffprobe -hide_banner -f lavfi -i "movie='Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4',entropy,scdet,signalstats=stat=tout+vrep+brng" -show_frames -show_streams -select_streams v -of json > "Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4.stat.json"

siti -of json -q -f "Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4" > "Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4.siti.json"
```

### Video segments coding and VQA calculation

Finally, segments are encoded with different CRF values to obtain the VMAF, PSNR and SSIM quality metrics using the ffmpeg + libvmaf v2.3.0 tool. Here are the commands for the first chunk of the video in 240 resolution and CRF set to 30 (results are saved in json file):

```bash
ffmpeg -hide_banner -y -i "Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4" -c:v libx264 -crf 30 -force_key_frames "expr:gte(t,n_forced*2)" -b:v 0 -threads 24 "Vlog_2160P-19f9_426x240_chunk0000_x264crf30.mp4"

ffmpeg -hide_banner -y -i "Vlog_2160P-19f9_426x240_chunk0000_x264crf30.mp4" -i "Vlog_2160P-19f9_1920x1080_chunk0000_crf0.mp4" -lavfi "[0:v]scale=1920:1080:flags=bicubic[distorted];[distorted][1:v]libvmaf=model='path=vmaf_v0.6.1.json':n_threads=24:log_fmt=json:log_path=Vlog_2160P-19f9_426x240_chunk0000_x264crf30.mp4.versus1080.vmaf.json:psnr=1:ssim=1:ms_ssim=1" -f null -
```

Note: file ```vmaf_v0.6.1.json``` is supposed to be in the same directory than chunks. If this file is in the default directory (```/usr/share/model/```), in the last command ```model='path=vmaf_v0.6.1.json'``` option is not necessary.

## Dataset handling

https://doi.org/10.6084/m9.figshare.c.6572563.v1

## Dataset statistical analysis and plots

[Follow this link](./webpages/README.md) to see the web pages displaying figures and statistical analyses conducted with all the datasets.
