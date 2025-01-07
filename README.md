# Prosody Diarization Tool
---
## Overview
The Prosody Diarization Tool is an **open-source** Python-based project designed to perform speaker diarization on audio files. It uses the `pyannote.audio` pipeline to identify and separate speakers in an audio file, and then splits and combines the audio segments for each speaker. The tool is particularly useful for analyzing interviews, meetings, or any multi-speaker audio recordings.

This project is divided into several stages:

1. **Folder Structure Setup**: Creates the necessary directories for storing audio files, diarization results, and speaker-specific audio segments.
2. **MP4 to WAV Conversion**: Converts MP4 video files to WAV audio files for processing.
3. **Speaker Diarization**: Uses the `pyannote.audio` pipeline to identify speakers and generate RTTM files.
4. **Splitting Audio**: Splits the audio file into segments based on the diarization results.
5. **Combining Speaker Segments**: Combines the segments for each speaker into a single audio file.
---
## Table of Contents
- [Installation](#installation)
- [Project Structure](#project-structure)
- [Usage](#usage)
  - [Folder Structure Setup](#folder-structure-setup)
  - [MP4 to WAV Conversion](#mp4-to-wav-conversion)
  - [Speaker Diarization](#speaker-diarization)
  - [Splitting Audio](#splitting-audio)
  - [Combining Speaker Segments](#combining-speaker-segments)

- [Dependencies](#dependencies)
- [License](#license)
- [Photos and Screenshots](#photos-and-screenshots)
---
## Installation
To use this tool, you need to install the required dependencies. Run the following command to install them:

```bash
pip install -r requirements.txt
```

Alternatively, you can install the dependencies manually:

```bash
pip install pyannote.audio moviepy pandas pydub torchaudio
```
---
## Project Structure
```
Database/
├── 0.Interviews/
│   ├── mp4/                # Input MP4 files
│   └── wav/                # Converted WAV files
├── 1.rttms/                # RTTM files from diarization
├── 2.Speaker_Parts/        # Split audio segments for each speaker
└── 3.Speaker_Combined/     # Combined audio files for each speaker
```
1. Stage 0: Identifies 2 speakers in the audio file using the pyannote.audio pipeline.

2. Stage 1: Identifies an unlimited number of speakers in the audio file if Stage 0 fails.

3. Stage 2: Identifies 3 speakers in the audio file if Stage 1 also fails.

--- 
## Usage
---
### Folder Structure Setup
Create the folder structure for storing the audio files, diarization results, and speaker-specific segments:

```python
import os

folders = [
    r".\Database\0.Interviews\mp4",
    r".\Database\0.Interviews\wav\stage_0",
    r".\Database\1.rttms\stage_0",
    r".\Database\2.Speaker_Parts\Stage_0",
    r".\Database\3.Speaker_Combined\Stage_0",
    r".\Database\0.Interviews\wav\stage_1",
    r".\Database\1.rttms\stage_1",
    r".\Database\2.Speaker_Parts\Stage_1",
    r".\Database\3.Speaker_Combined\Stage_1"
]

for folder in folders:
    os.makedirs(folder, exist_ok=True)
```

### MP4 to WAV Conversion
Convert MP4 files to WAV format for processing:
---
```python
from moviepy.editor import VideoFileClip

def convert_video_to_audio(input_video, output_audio):
    video_clip = VideoFileClip(input_video)
    audio_clip = video_clip.audio
    audio_clip.write_audiofile(output_audio, codec='pcm_s16le', bitrate='16k')

input_dir = r".\Database\0.Interviews\mp4"
output_dir = r".\Database\0.Interviews\wav\stage_0"

for file in os.listdir(input_dir):
    if file.endswith(".mp4"):
        input_video = os.path.join(input_dir, file)
        output_audio = os.path.join(output_dir, os.path.splitext(file)[0] + ".wav")
        convert_video_to_audio(input_video, output_audio)
```
---
### Speaker Diarization
Perform speaker diarization using the `pyannote.audio` pipeline:

```python
from pyannote.audio import Pipeline

pipeline = Pipeline.from_pretrained(
    "pyannote/speaker-diarization-3.1",
    use_auth_token="your_huggingface_token"
)

input_dir = r".\Database\0.Interviews\wav\stage_0"
output_dir = r".\Database\1.rttms\stage_0"

for file_name in os.listdir(input_dir):
    if file_name.endswith(".wav"):
        wav_file_path = os.path.join(input_dir, file_name)
        diarization = pipeline(wav_file_path, num_speakers=2)
        rttm_file_path = os.path.join(output_dir, f"{os.path.splitext(file_name)[0]}.rttm")
        with open(rttm_file_path, "w") as rttm_file:
            diarization.write_rttm(rttm_file)
```
---
### Splitting Audio
Split the audio file into segments for each speaker:

```python
from pydub import AudioSegment

def split_audio(input_path, output_path, start, duration, filename):
    sound = AudioSegment.from_wav(input_path)
    split_sound = sound[start*1000:(start+duration)*1000]
    split_sound.export(os.path.join(output_path, filename), format="wav")

df = combined_df  # DataFrame from the RTTM files
input_dir = r".\Database\0.Interviews\wav\Stage_0"
output_dir = r".\Database\2.Speaker_Parts\Stage_0"

for index, row in df.iterrows():
    interview_id = row['Interview_ID']
    speaker_id = row['Speaker_ID']
    start = row['Start']
    duration = row['Duration']
    filename = f"{interview_id}_{speaker_id}_{index}.wav"
    input_path = os.path.join(input_dir, f"{interview_id}.wav")
    output_path = os.path.join(output_dir, speaker_id)
    os.makedirs(output_path, exist_ok=True)
    split_audio(input_path, output_path, start, duration, filename)
```
---
### Combining Speaker Segments
Combine the segments for each speaker into a single audio file:

```python
from pydub import AudioSegment

def concatenate_wav_files_with_gap(directory, output, gap_duration=0.1):
    wave_files = {}
    for filename in os.listdir(directory):
        if filename.endswith(".wav"):
            prefix = '_'.join(filename.split('_')[:3])
            if prefix in wave_files:
                wave_files[prefix].append(filename)
            else:
                wave_files[prefix] = [filename]

    for prefix, files in wave_files.items():
        files.sort(key=lambda x: int(x.split('_')[-1].split('.')[0]))
        combined = AudioSegment.silent(duration=0)
        for file in files:
            current_segment = AudioSegment.from_wav(os.path.join(directory, file))
            combined += current_segment + AudioSegment.silent(duration=gap_duration * 1000)
        output_filename = os.path.join(output, f"{prefix}_{os.path.basename(directory)}.wav")
        combined.export(output_filename, format="wav")

speakers = [f"SPEAKER_{str(i).zfill(2)}" for i in range(2)]
stage = "Stage_0"

for speaker in speakers:
    directory = rf".\Database\2.Speaker_Parts\{stage}\{speaker}"
    output = rf".\Database\3.Speaker_Combined\{stage}\{speaker}"
    os.makedirs(output, exist_ok=True)
    concatenate_wav_files_with_gap(directory, output, gap_duration=0.1)
```
---  
## Dependencies
- `pyannote.audio`
- `moviepy`
- `pandas`
- `pydub`
- `torchaudio`
---
## License
This project is licensed under the MIT License. See the LICENSE file for details.

## Photos and Screenshots
- **Folder Structure**:
  
![image](https://github.com/user-attachments/assets/4e3319eb-ece3-43d2-b1c2-ce4abbd85cf4)

- **Diarization Results**: the RTTM file of the diarization output.

  ![image](https://github.com/user-attachments/assets/67465c3f-0d78-4fb7-8eaa-a6d5dbd577ec)

- **Audio Segments**: the split audio files in the `\Database\2.Speaker_Parts\Stage_0\SPEAKER_00` folder.
![image](https://github.com/user-attachments/assets/f6069852-6e22-4063-ac07-ebeb0c574133)

- **Combined Audio**: the combined audio files in the `\Database\3.Speaker_Combined\Stage_0\SPEAKER_00` folder.

![image](https://github.com/user-attachments/assets/1d28f4f7-1ee0-4d3d-9419-c9dcd669ff7a)

---

## Acknowledgments

This project, part of the Prosody in Mental Health initiative, is developed at **Reutlingen University**.

- **Author:** Mohamad Eyad Alkostantini   
- **Supervisors:** Bakir Hadzic, Parvez Mohammed, (ViSiR, Actimi GmbH, Neuronation (Synaptikon GmbH) and PFH Göttingen)
- **Professor:** Prof. Matthias Rätsch 

We gratefully acknowledge their guidance and support throughout the project.

---

## Contact
For questions or support, please reach out to [Mohamad_Eyad.Alkostantini@Student.Reutlingen-University.DE].
