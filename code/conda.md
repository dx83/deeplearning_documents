# conda 새 파이썬 환경 생성
conda create -n 환경이름 python=3.11
conda create -n 환경이름 python=3.11 설치패키지 설치패키지
conda activate 환경이름

# openai-whisper 환경 생성
```
conda create -n whisper python=3.11 ffmpeg -c conda-forge
conda activate whisper
pip install torch
pip install -U openai-whisper
```
