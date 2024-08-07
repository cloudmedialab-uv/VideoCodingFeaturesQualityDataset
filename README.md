# FEATURES AND QUALITY METRICS DATASETS FOR VIDEO CODING IN DASH

This repository is related to the paper "Features and Quality Metrics Datasets for Video Coding in DASH" submitted to the journal "Scientific Data" and comprises sample Bash and Python source code. The Bash scripts are employed to extract features and compute VQA metrics from 4k videos, with the objective of creating datasets that are publicly available on figshare. The sample Python source code show how to combine and process the datasets. Finally, we present web pages displaying figures and statistical analyses conducted with all the datasets.

# Table of contents
1. [Sample comands used to extract features and compute VQA metrics](#sample-comands-used-to-extract-features-and-compute-vqa-metrics)
2. [Dataset handling](#dataset-handling)
3. [Dataset statistical analysis and plots](#dataset-statistical-analysis-and-plots)

<!--## Cite
This work has been published in xxx:

If you use part of or all the datasets, please cite that paper.
-->

## Sample comands used to extract features and compute VQA metrics

In the following we assume that you have the [```ffmpeg```](https://ffmpeg.org) and [```siti```](https://github.com/slhck/siti) tools installed. The steps to extract features and to compute VQA metrics for one 4k video (we will use a sample video file ```Vlog_2160P-19f9.mkv``` from the [```YouTube User Generated Content dataset```](https://media.withyoutube.com)) are: 

### 1) Download sample video

First we obtain the file from the UGC dataset:

```bash
wget https://storage.googleapis.com/ugc-dataset/original_videos/Vlog/2160P/Vlog_2160P-19f9.mkv
```

### 2) Video homogenization

Taking into account the diverse range of containers, chroma sampling, and bit depths of the videos downloaded from all the databases referenced in paper, the next step involve homogenizing these characteristics:

```bash
ffmpeg -hide_banner -i "Vlog_2160P-19f9.mkv" -an -c:v libx264 -crf 0 -preset ultrafast -pix_fmt yuv420p "Vlog_2160P-19f9_3840x2160_crf0.mp4"
```

### 3) Video temporal splitting and spatial downscaling

After the standardisation process, each video is divided into 2-seconds chunks with vertical resolutions of 240, 360, 480, 720, 1080, 1440 and 4K pixels:

```bash
ffmpeg -hide_banner -i "Vlog_2160P-19f9_3840x2160_crf0.mp4" -filter_complex "[0:v]yadif,split=7[out1][out2][out3][out4][out5][out6][out7]" -map "[out1]" -s 3840x2160 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_3840x2160_chunk%4d_crf0.mp4" -map "[out2]" -s 2560x1440 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_2560x1440_chunk%4d_crf0.mp4" -map "[out3]" -s 1920x1080 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_1920x1080_chunk%4d_crf0.mp4" -map "[out4]" -s 1280x720 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_1280x720_chunk%4d_crf0.mp4" -map "[out5]" -s 854x480 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_854x480_chunk%4d_crf0.mp4" -map "[out6]" -s 640x360 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_640x360_chunk%4d_crf0.mp4" -map "[out7]" -s 426x240 -sws_flags lanczos -c:v libx264 -crf 0 -preset ultrafast -f segment -segment_time 2 -force_key_frames "expr:gte(t,n_forced * 2)" -reset_timestamps 1 "Vlog_2160P-19f9_426x240_chunk%4d_crf0.mp4"
```

### 4) Features extraction

By utilizing the ffprobe tool with the signalstats and entropy filters, as well as employing the Python siti package v1.4.5 in accordance with 2021 ITU-T Recommendation P.910, we extract a collection of features from all the chunks created in the previous step and save them into json files. The commands for the first chunk of the video in 240 resolution are:

```bash
ffprobe -hide_banner -f lavfi -i "movie='Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4',entropy,scdet,signalstats=stat=tout+vrep+brng" -show_frames -show_streams -select_streams v -of json > "Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4.stat.json"

siti -of json -q -f "Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4" > "Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4.siti.json"
```

### 5) Video segments coding and VQA calculation

Finally, segments are encoded with different CRF values to obtain the VMAF, PSNR and SSIM quality metrics using the ffmpeg + libvmaf v2.3.0 tool. 

For resolution equal or lower than 1080, coded segments have to be upscaled to 1080 resolution for VMAF calculation.

For example, the commands for the first chunk of the video in 240 resolution and CRF set to 30 (results are saved in json file) are:

```bash
ffmpeg -hide_banner -y -i "Vlog_2160P-19f9_426x240_chunk0000_crf0.mp4" -c:v libx264 -crf 30 -force_key_frames "expr:gte(t,n_forced*2)" -b:v 0 -threads 24 "Vlog_2160P-19f9_426x240_chunk0000_x264crf30.mp4"

ffmpeg -hide_banner -y -i "Vlog_2160P-19f9_426x240_chunk0000_x264crf30.mp4" -i "Vlog_2160P-19f9_1920x1080_chunk0000_crf0.mp4" -lavfi "[0:v]scale=1920:1080:flags=bicubic[distorted];[distorted][1:v]libvmaf=model='path=vmaf_v0.6.1.json':n_threads=24:log_fmt=json:log_path=Vlog_2160P-19f9_426x240_chunk0000_x264crf30.mp4.versus1080.vmaf.json:psnr=1:ssim=1:ms_ssim=1" -f null -
```

For 1440 and 4K resolutions, coded segments have to be upscaled to 4K resolution for VMAF calculation.

For example, the commands for the first chunk of the video in 1440 resolution and CRF set to 23 (results are saved in json file) are:

```bash
ffmpeg -hide_banner -y -i "Vlog_2160P-19f9_2560x1440_chunk0000_crf0.mp4" -c:v libx264 -crf 23 -force_key_frames "expr:gte(t,n_forced*2)" -b:v 0 -threads 24 "Vlog_2160P-19f9_2560x1440_chunk0000_x264crf23.mp4"

ffmpeg -hide_banner -y -i "Vlog_2160P-19f9_2560x1440_chunk0000_x264crf23.mp4" -i "Vlog_2160P-19f9_3840x2160_chunk0000_crf0.mp4" -lavfi "[0:v]scale=3840:2160:flags=bicubic[distorted];[distorted][1:v]libvmaf=model='path=vmaf_4k_v0.6.1.json':n_threads=24:log_fmt=json:log_path=Vlog_2160P-19f9_2560x1440_chunk0000_x264crf23.mp4.versus2160.vmaf.json:psnr=1:ssim=1:ms_ssim=1" -f null -
```

Note: files ```vmaf_v0.6.1.json``` and ```vmaf_4k_v0.6.1.json``` are supposed to be in the same directory than chunks. If these files are in the default directory (```/usr/share/model/```), in VMAF calculation command ```model='path=vmaf_v0.6.1.json'``` and ```model='path=vmaf_4k_v0.6.1.json'``` options are not necessary.

## Dataset handling

### 1) Downloading datasets

<!-- >> :exclamation: Links will be updated upon paper acceptance-->

Datasets can be downloaded from https://doi.org/10.6084/m9.figshare.c.6764328.v1.

For example if we want to download the features dataset of segments with a resolution of 720, we can download the file directly from the above link or we can execute:

```bash
wget -O Features_720.csv https://springernature.figshare.com/ndownloader/files/41742810
```

### 2) Combining datasets with matching resolution

Once we have downloaded the datasets, it is possible to combine the features datasets with the quality metrics datasets using the [```pandas```](https://pandas.pydata.org) tool in Python.

Let's consider, for example, that we want to relate the features and quality metrics obtained from segments with a resolution of 1080. The following script would need to be executed:

```python
import pandas as pd

dfFeatures = pd.read_csv('Features_1080.csv')
dfQuality = pd.read_csv('Quality_1080.csv')
df = pd.merge(dfFeatures, dfQuality)
```

A table with 153277 rows and 49 columns is obtained, which relates the features of a segment with the metrics obtained for each CRF used in the encoding of that segment.

The next table shows the first 5 rows of the combined table displaying the features of the **first** segment (corresponding to chunk 0) of the video **AncientThought** related to the metrics obtained with **CRF values 7, 8, 9, 10, and 11**:

|    | file           |   chunk |   res |   Ymean |    Ystd |    Ykurt |    Yskew |   Umean |    Ustd |    Ukurt |    Uskew |   Vmean |    Vstd |    Vkurt |     Vskew |   SATmean |   SATstd |   SATkurt |   SATskew |   HUEmean |   HUEstd |   HUEkurt |   HUEskew |   nEYmean |   nEYstd |   nEYkurt |   nEYskew |   nEUmean |   nEUstd |   nEUkurt |   nEUskew |   nEVmean |   nEVstd |   nEVkurt |   nEVskew |   sceneChange |     fr |   TImean |   TIstd |    TIkurt |   TIskew |   SImean |   SIstd |    SIkurt |   SIskew |   crf |   vmafmean |   psnrmean |   ssimmean |
|---:|:---------------|--------:|------:|--------:|--------:|---------:|---------:|--------:|--------:|---------:|---------:|--------:|--------:|---------:|----------:|----------:|---------:|----------:|----------:|----------:|---------:|----------:|----------:|----------:|---------:|----------:|----------:|----------:|---------:|----------:|----------:|----------:|---------:|----------:|----------:|--------------:|-------:|---------:|--------:|----------:|---------:|---------:|--------:|----------:|---------:|------:|-----------:|-----------:|-----------:|
|  0 | AncientThought |       0 |  1080 | 40.5577 | 16.7694 | -1.19587 | 0.269313 | 122.584 | 5.10596 | -1.19084 | -0.17363 | 131.732 | 2.66463 | -1.64651 | -0.134091 |   7.74845 |  5.64675 |  -1.30153 |  0.105061 |   182.221 |  19.7049 |   -1.5566 |  0.363657 |  0.549891 | 0.181782 |  -1.59896 | -0.346912 |  0.436096 | 0.143368 |  -1.64158 | -0.312752 |  0.376922 | 0.131878 |  -1.59926 |    -0.491 |             0 | 23.976 |   16.267 |   11.88 | -0.530314 | 0.302004 |    8.131 |   2.201 | 0.0896343 | -1.05176 |     7 |    98.291  |    55.5511 |   0.999819 |
|  1 | AncientThought |       0 |  1080 | 40.5577 | 16.7694 | -1.19587 | 0.269313 | 122.584 | 5.10596 | -1.19084 | -0.17363 | 131.732 | 2.66463 | -1.64651 | -0.134091 |   7.74845 |  5.64675 |  -1.30153 |  0.105061 |   182.221 |  19.7049 |   -1.5566 |  0.363657 |  0.549891 | 0.181782 |  -1.59896 | -0.346912 |  0.436096 | 0.143368 |  -1.64158 | -0.312752 |  0.376922 | 0.131878 |  -1.59926 |    -0.491 |             0 | 23.976 |   16.267 |   11.88 | -0.530314 | 0.302004 |    8.131 |   2.201 | 0.0896343 | -1.05176 |     8 |    98.1204 |    54.8735 |   0.999778 |
|  2 | AncientThought |       0 |  1080 | 40.5577 | 16.7694 | -1.19587 | 0.269313 | 122.584 | 5.10596 | -1.19084 | -0.17363 | 131.732 | 2.66463 | -1.64651 | -0.134091 |   7.74845 |  5.64675 |  -1.30153 |  0.105061 |   182.221 |  19.7049 |   -1.5566 |  0.363657 |  0.549891 | 0.181782 |  -1.59896 | -0.346912 |  0.436096 | 0.143368 |  -1.64158 | -0.312752 |  0.376922 | 0.131878 |  -1.59926 |    -0.491 |             0 | 23.976 |   16.267 |   11.88 | -0.530314 | 0.302004 |    8.131 |   2.201 | 0.0896343 | -1.05176 |     9 |    97.8994 |    54.2277 |   0.999724 |
|  3 | AncientThought |       0 |  1080 | 40.5577 | 16.7694 | -1.19587 | 0.269313 | 122.584 | 5.10596 | -1.19084 | -0.17363 | 131.732 | 2.66463 | -1.64651 | -0.134091 |   7.74845 |  5.64675 |  -1.30153 |  0.105061 |   182.221 |  19.7049 |   -1.5566 |  0.363657 |  0.549891 | 0.181782 |  -1.59896 | -0.346912 |  0.436096 | 0.143368 |  -1.64158 | -0.312752 |  0.376922 | 0.131878 |  -1.59926 |    -0.491 |             0 | 23.976 |   16.267 |   11.88 | -0.530314 | 0.302004 |    8.131 |   2.201 | 0.0896343 | -1.05176 |    10 |    97.682  |    53.6968 |   0.999665 |
|  4 | AncientThought |       0 |  1080 | 40.5577 | 16.7694 | -1.19587 | 0.269313 | 122.584 | 5.10596 | -1.19084 | -0.17363 | 131.732 | 2.66463 | -1.64651 | -0.134091 |   7.74845 |  5.64675 |  -1.30153 |  0.105061 |   182.221 |  19.7049 |   -1.5566 |  0.363657 |  0.549891 | 0.181782 |  -1.59896 | -0.346912 |  0.436096 | 0.143368 |  -1.64158 | -0.312752 |  0.376922 | 0.131878 |  -1.59926 |    -0.491 |             0 | 23.976 |   16.267 |   11.88 | -0.530314 | 0.302004 |    8.131 |   2.201 | 0.0896343 | -1.05176 |    11 |    97.4401 |    53.1875 |   0.999586 |

### 3) Shape of the combined tables according to the resolution

If we want to determine the number of rows we will obtain when combining the feature datasets with the metric datasets for each resolution, we can execute the following script:

```python
import pandas as pd

for res in [240, 360, 480, 720, 1080, 1440, '4kmax']:
    dfFeat = pd.read_csv(f'Features_{res}.csv')
    dfQual = pd.read_csv(f'Quality_{res}.csv')
    df = pd.merge(dfFeat, dfQual)
    # We prevent the loss of the combined tables just generated
    df.to_csv(f'Merged_{res}.csv', index=False)
    print(f'{res}: Features {dfFeat.shape} - Quality {dfQual.shape} - Merged {df.shape}')
```

The result obtained by running this code is:

```
240: Features (4065, 45) - Quality (154544, 7) - Merged (154544, 49)
360: Features (4065, 45) - Quality (168083, 7) - Merged (168083, 49)
480: Features (4065, 45) - Quality (173729, 7) - Merged (173729, 49)
720: Features (4065, 45) - Quality (172451, 7) - Merged (172451, 49)
1080: Features (4065, 45) - Quality (153277, 7) - Merged (153277, 49)
1440: Features (4065, 45) - Quality (160791, 7) - Merged (160791, 49)
4kmax: Features (4065, 45) - Quality (133516, 7) - Merged (133516, 49)
```

<!-- In the case of using datasets where entries with NaN values have been removed, the result would be different.

This is the script:

```python
import pandas as pd

for res in [240, 360, 480, 720, 1080, 1440, '4kmax']:
    dfFeat = pd.read_csv(f'Features_{res}_withoutNaN.csv')
    dfQual = pd.read_csv(f'Quality_{res}_withoutNaN.csv')
    df = pd.merge(dfFeat, dfQual)
    # We prevent the loss of the combined tables just generated
    df.to_csv(f'Merged_{res}_withoutNaN.csv', index=False)
    print(f'{res}: Features {dfFeat.shape} - Quality {dfQual.shape} - Merged {df.shape}')
```

And this the new outcome:

```
240: Features (3929, 45) - Quality (149123, 7) - Merged (149123, 49)
360: Features (3929, 45) - Quality (162205, 7) - Merged (162205, 49)
480: Features (3928, 45) - Quality (167625, 7) - Merged (167625, 49)
720: Features (3928, 45) - Quality (166294, 7) - Merged (166294, 49)
1080: Features (3928, 45) - Quality (147112, 7) - Merged (147112, 49) 
```
-->

### 4) Combining all datasets into a single table

It is possible to create a unified table that combines all datasets from all resolutions with the following script (example for datasets with NaN values):

```python
dfslist = []
for res in [240, 360, 480, 720, 1080, 1440, '4kmax']:
    dfFeat = pd.read_csv(f'Features_{res}.csv')
    dfQual = pd.read_csv(f'Quality_{res}.csv')
    df = pd.merge(dfFeat, dfQual)
    dfslist.append(df)

dffinal = pd.concat(dfslist, ignore_index=True)
```

The resulting final table in this case comprises 1,116,391 rows and 49 columns.

## Dataset statistical analysis and plots

[Follow this link](./webpages/README.md) to see the web pages displaying figures and statistical analyses conducted with all the datasets.
