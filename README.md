# voice-classification
python3 run_wav2vec_clf.py \
    --pooling_mode="mean" \
    --model_name_or_path="lighteternal/wav2vec2-large-xlsr-53-greek" \
    --model_mode="wav2vec2" \ # or you can use hubert
    --output_dir=/path/to/output \
    --cache_dir=/path/to/cache/ \
    --train_file=/path/to/train.csv \
    --validation_file=/path/to/dev.csv \
    --test_file=/path/to/test.csv \
    --per_device_train_batch_size=4 \
    --per_device_eval_batch_size=4 \
    --gradient_accumulation_steps=2 \
    --learning_rate=1e-4 \
    --num_train_epochs=5.0 \
    --evaluation_strategy="steps"\
    --save_steps=100 \
    --eval_steps=100 \
    --logging_steps=100 \
    --save_total_limit=2 \
    --do_eval \
    --do_train \
    --fp16 \
    --freeze_feature_extractor



## code for voice classification based on ML (PREDICTIIONS).

import torch
import torch.nn as nn
import torch.nn.functional as F
import torchaudio
from transformers import AutoConfig, Wav2Vec2FeatureExtractor
from src.models import Wav2Vec2ForSpeechClassification, HubertForSpeechClassification

model_name_or_path = "path/to/your-pretrained-model"

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
config = AutoConfig.from_pretrained(model_name_or_path)
feature_extractor = Wav2Vec2FeatureExtractor.from_pretrained(model_name_or_path)
sampling_rate = feature_extractor.sampling_rate

# for wav2vec
model = Wav2Vec2ForSpeechClassification.from_pretrained(model_name_or_path).to(device)

# for hubert
model = HubertForSpeechClassification.from_pretrained(model_name_or_path).to(device)


def speech_file_to_array_fn(path, sampling_rate):
    speech_array, _sampling_rate = torchaudio.load(path)
    resampler = torchaudio.transforms.Resample(_sampling_rate, sampling_rate)
    speech = resampler(speech_array).squeeze().numpy()
    return speech


def predict(path, sampling_rate):
    speech = speech_file_to_array_fn(path, sampling_rate)
    inputs = feature_extractor(speech, sampling_rate=sampling_rate, return_tensors="pt", padding=True)
    inputs = {key: inputs[key].to(device) for key in inputs}

    with torch.no_grad():
        logits = model(**inputs).logits

    scores = F.softmax(logits, dim=1).detach().cpu().numpy()[0]
    outputs = [{"Emotion": config.id2label[i], "Score": f"{round(score * 100, 3):.1f}%"} for i, score in
               enumerate(scores)]
    return outputs


path = "/path/to/disgust.wav"
outputs = predict(path, sampling_rate)    
