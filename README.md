# Stabilizing Timestamps for Whisper

This library modifies [Whisper](https://github.com/openai/whisper) to produce more reliable timestamps and extends its functionality.

https://user-images.githubusercontent.com/28970749/225826345-ef7115db-51e4-4b23-aedd-069389b8ae43.mp4

* [Setup](#setup)
* [Usage](#usage)
  * [Transcribe](#transcribe)
  * [Output](#output)
  * [Alignment](#alignment)
    * [Adjustments](#adjustments)
  * [Refinement](#refinement)
  * [Regrouping Words](#regrouping-words)
  * [Editing](#editing)
  * [Locating Words](#locating-words)
  * [Silence Suppression](#silence-suppression)
  * [Tips](#tips)
  * [Visualizing Suppression](#visualizing-suppression)
  * [Encode Comparison](#encode-comparison)
  * [Use with any ASR](#any-asr)
* [Quick 1.X → 2.X Guide](#quick-1x--2x-guide)

## Setup
```
pip install -U stable-ts
```

To install the latest commit:
```
pip install -U git+https://github.com/jianfch/stable-ts.git
```

## Usage

### Transcribe

```python
import stable_whisper
model = stable_whisper.load_model('base')
result = model.transcribe('audio.mp3')
result.to_srt_vtt('audio.srt')
```
<details>
<summary>CLI</summary>

```commandline
stable-ts audio.mp3 -o audio.srt
```

</details>

Docstrings: 
[load_model()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/whisper_word_level.py#L1041-L1071), 
[transcribe()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/whisper_word_level.py#L77-L220),
[transcribe_minimal()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/whisper_word_level.py#L680-L755)

<details>
<summary>faster-whisper</summary>

Use with [faster-whisper](https://github.com/guillaumekln/faster-whisper):
```python
model = stable_whisper.load_faster_whisper('base')
result = model.transcribe_stable('audio.mp3')
```
```commandline
stable-ts audio.mp3 -o audio.srt -fw
```
Docstring: 
[load_faster_whisper()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/whisper_word_level.py#L795-L812),
[transcribe_stable()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/whisper_word_level.py#L880-L964)

</details>

### Output
Stable-ts supports various text output formats.
```python
result.to_srt_vtt('audio.srt') #SRT
result.to_srt_vtt('audio.vtt') #VTT
result.to_ass('audio.ass') #ASS
result.to_tsv('audio.tsv') #TSV
```
Docstrings: 
[to_srt_vtt()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/text_output.py#L266-L308),
[to_ass()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/text_output.py#L412-L465),
[to_tsv()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/text_output.py#L340-L378)
[to_txt()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/text_output.py#L534-L568)
[save_as_json()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/text_output.py#L587-L597)
<br /><br />
There are word-level and segment-level timestamps. All output formats support them. 
They also support will both levels simultaneously except TSV. 
By default, `segment_level` and `word_level` are both `True` for all the formats that support both simultaneously.<br /><br />
Examples in VTT.

Default: `segment_level=True` + `word_level=True`
<details>
<summary>CLI</summary>

`--segment_level true` + `--word_level true`

</details>

```
00:00:07.760 --> 00:00:09.900
But<00:00:07.860> when<00:00:08.040> you<00:00:08.280> arrived<00:00:08.580> at<00:00:08.800> that<00:00:09.000> distant<00:00:09.400> world,
```

`segment_level=True`  + `word_level=False`
```
00:00:07.760 --> 00:00:09.900
But when you arrived at that distant world,
```

`segment_level=False` + `word_level=True`
```
00:00:07.760 --> 00:00:07.860
But

00:00:07.860 --> 00:00:08.040
when

00:00:08.040 --> 00:00:08.280
you

00:00:08.280 --> 00:00:08.580
arrived

...
```

#### JSON
The result can also be saved as a JSON file to preserve all the data for future reprocessing. 
This is useful for testing different sets of postprocessing arguments without the need to redo inference.

```python
result.save_as_json('audio.json')
```
<details>
<summary>CLI</summary>

```commandline
stable-ts audio.mp3 -o audio.json
```

</details>

Processing JSON file of the results into SRT.
```python
result = stable_whisper.WhisperResult('audio.json')
result.to_srt_vtt('audio.srt')
```
<details>
<summary>CLI</summary>

```commandline
stable-ts audio.json -o audio.srt
```

</details>

### Alignment
Audio can be aligned/synced with plain text on word-level.
```python
text = 'Machines thinking, breeding. You were to bear us a new, promised land.'
result = model.align('audio.mp3', text, language='en')
```
When the text is correct but the timestamps need more work, 
`align()` is a faster alternative for testing various settings/models.
```python
new_result = model.align('audio.mp3', result, language='en')
```
<details>
<summary>CLI</summary>

```commandline
stable-ts audio.mp3 --align text.txt --language en
```
`--align` can also a JSON file of a result 

</details>

Docstring:
[align()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/alignment.py#L60-L167)

#### Adjustments
Timestamps are adjusted after the model predicts them. 
When `suppress_silence=True` (default), `transcribe()`/`transcribe_minimal()`/`align()` adjust based on silence/non-speech. 
The timestamps can be further adjusted base on another result with `adjust_by_result()`, 
which acts as a logical AND operation for the timestamps of both results, further reducing duration of each word.
Note: both results are required to have word timestamps and matching words.
```python
# the adjustments are in-place for `result`
result.adjust_by_result(new_result)
```
Docstring:
[adjust_by_result()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L879-L890)

### Refinement
Timestamps can be further improved with `refine()`.
This method iteratively mutes portions of the audio based on current timestamps 
then compute the probabilities of the tokens. 
Then by monitoring the fluctuation of the probabilities, it tries to find the most precise timestamps. 
"Most precise" in this case means the latest start and earliest end for the word 
such that it still meets the specified conditions.
```python
model.refine('audio.mp3', result)
```
<details>
<summary>CLI</summary>

```commandline
stable-ts audio.mp3 --refine -o audio.srt
```
Input can also be JSON file of a result. 
```commandline
stable-ts result.json --refine -o audio.srt --refine_option "audio=audio.mp3"
```

</details>

Docstring:
[refine()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/alignment.py#L459-L530)


### Regrouping Words
Stable-ts has a preset for regrouping words into different segments with more natural boundaries. 
This preset is enabled by `regroup=True` (default). 
But there are other built-in [regrouping methods](#regrouping-methods) that allow you to customize the regrouping algorithm. 
This preset is just a predefined combination of those methods.

https://user-images.githubusercontent.com/28970749/226504985-3d087539-cfa4-46d1-8eb5-7083f235b429.mp4

```python
# The following results are all functionally equivalent:
result0 = model.transcribe('audio.mp3', regroup=True) # regroup is True by default
result1 = model.transcribe('audio.mp3', regroup=False)
(
    result1
    .clamp_max()
    .split_by_punctuation([('.', ' '), '。', '?', '？', (',', ' '), '，'])
    .split_by_gap(.5)
    .merge_by_gap(.3, max_words=3)
    .split_by_punctuation([('.', ' '), '。', '?', '？'])
)
result2 = model.transcribe('audio.mp3', regroup='cm_sp=.* /。/?/？/,* /，_sg=.5_mg=.3+3_sp=.* /。/?/？')

# To undo all regrouping operations:
result0.reset()
```
Any regrouping algorithm can be expressed as a string. Please feel free share your strings [here](https://github.com/jianfch/stable-ts/discussions/162)
#### Regrouping Methods
- [regroup()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1886-L1943)
- [split_by_gap()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1114-L1130)
- [split_by_punctuation()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1184-L1206)
- [split_by_length()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1302-L1333)
- [split_by_duration()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1361-L1390)
- [merge_by_gap()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1145-L1166)
- [merge_by_punctuation()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1235-L1256)
- [merge_all_segments()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1267-L1274)
- [clamp_max()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1415-L1437)
- [lock()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1480-L1502)

### Editing
The editing methods in stable-ts can be chained with [Regrouping Methods](#regrouping-methods) and used in `regroup()`.

Remove specific instances words or segments:
```python
# Remove first word of the first segment:
first_word = result[0][0]
result.remove_word(first_word)
# This following is also does the same:
del result[0][0]

# Remove the last segment:
last_segment = result[-1]
result.remove_segment(last_segment)
# This following is also does the same:
del result[-1]
```
Docstrings:
[remove_word()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1544-L1560),
[remove_segment()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1586-L1602)


Removing repetitions:
```python
# Example 1: "This is is is a test." -> "This is a test."
# The following removes the last two " is":
result.remove_repetition(1)

# Example 2: "This is is is a test this is a test." -> "This is a test."
# The following removes the second " is" and third " is", then remove the last "this is a test"
# The first parameter `max_words` is `4` because "this is a test" consists 4 words
result.remove_repetition(4)
```
Docstring:
[remove_repetition()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1626-L1648)

Removing specific word(s) by string content:
```python
# Remove all " ok" from " ok ok this is a test."
result.remove_words_by_str('ok')

# Remove all " ok" and " Um..." from " ok this is a test. Um..."
result.remove_words_by_str(['ok', 'um'])
```
Docstring:
[remove_words_by_str()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1714-L1741)

Filling in segment gaps:
```python
# result0:             [" How are you?"] [" I'm good."]                     [" Good!"]
# result1: [" Hello!"] [" How are you?"]                [" How about you?"] [" Good!"]
result0.fill_in_gaps(result1)
# After filling in the gaps in `result0` with contents in `result1`:
# result0: [" Hello!"] [" How are you?"] [" I'm good."] [" How about you?"] [" Good!"]
```
Docstring:
[fill_in_gaps()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1786-L1812)

### Locating Words
There are two ways to locate words. 
The first way is by approximating time at which the words are spoken 
then transcribing a few seconds around the approximated time.
This also the faster way for locating words.
```python
matches = model.locate('audio.mp3', 'are', language='en', count=0)
for match in matches:
    print(match.to_display_str())
# verbose=True does the same thing as this for-loop.
```
Docstring:
[locate()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/alignment.py#L839-L935)

<details>
<summary>CLI</summary>

```
stable-ts audio.mp3 --locate "are" --language en -to "count=0"
```

</details>

The second way allows you to locate words with regular expression,
but it requires the audio to be fully transcribed first. 
```python
result = model.transcribe('audio.mp3')
# Find every sentence that contains "and"
matches = result.find(r'[^.]+and[^.]+\.')
# print the all matches if there are any
for match in matches:
  print(f'match: {match.text_match}\n'
        f'text: {match.text}\n'
        f'start: {match.start}\n'
        f'end: {match.end}\n')
  
# Find the word before and after "and" in the matches
matches = matches.find(r'\s\S+\sand\s\S+')
for match in matches:
  print(f'match: {match.text_match}\n'
        f'text: {match.text}\n'
        f'start: {match.start}\n'
        f'end: {match.end}\n')
```
Docstring: 
[find()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/result.py#L1999-L1015)

### Silence Suppression
While the timestamps predicted by Whisper are generally accurate, 
it sometimes predicts the start of a word way before the word is spoken 
or the end of a word long after the word has been spoken.
This is where "silence suppression" helps. It is enabled by default (`suppress_silence=True`).
The idea is to adjust the timestamps based on the timestamps of non-speech portions of the audio.
![silence_suppresion0](./silence_suppresion0.png)
*Note: In V1, "silence suppressing" refers to the process of suppressing timestamp tokens of the silent portions, 
but changed to postprocessing adjustments in V2, which allows stable-ts to be used with other ASR models. 
The timestamp token suppression feature is disabled by default, but can still be enabled with `suppress_ts_tokens=True`*

By default, stable-ts determines the non-speech timestamps based on 
how loud a section of the audio is relative to the neighboring sections. 
This method is most effective for cases, where the speech is significantly louder than the background noise.
The other method is to use [Silero VAD](https://github.com/snakers4/silero-vad) (enabled with `vad=True`). 
To visualize the differences between non-VAD and VAD, see [Visualizing Suppression](#visualizing-suppression).

Besides the parameters for non-speech detection sensitivity (see [Visualizing Suppression](#visualizing-suppression)), 
the following parameters are used to combat inaccurate non-speech detection.<br>
`min_word_dur` is the shortest duration each word is allowed from adjustments.<br>
`nonspeech_error` is the relative error of the non-speech that appears in between a word.<br>
*Note: Before 2.14, `nonspeech_error` was not available, 
and `min_word_dur` prevented any adjustments that resulted in word duration shorter than `min_word_dur`.*

For the following example, `min_word_dur=0.5` (default: 0.1) and `nonspeech_error=0.3` (default: 0.3).
![silence_suppresion1](./silence_suppresion1.png) 
`nonspeech_error=0.3` allows each non-speech section to be treated 1.3 times their actual duration.
Either from the start of the corresponding word to the end of the non-speech 
or from the start of the non-speech to the end of the corresponding word.
In the case that both conditions are met, the shorter one is used.
Or if both are equal, then the start of the non-speech to the end of the word is used.<br>
The second non-speech from 1.375s to 1.75s is ignored for 'world.' because it failed both conditions.<br>
The first word, 'Hello', satisfies only the former condition from 0s to 0.625, thus the new start for 'Hello'
would be 0.625s. However, `min_word_dur=0.5` requires the resultant duration to be at least 0.5s. 
As a result, the start of 'Hello' is changed to 0.375s instead of 0.625s. 

### Tips
- do not disable word timestamps with `word_timestamps=False` for reliable segment timestamps
- use `vad=True` for more accurate non-speech detection
- use `demucs=True` to isolate vocals with [Demucs](https://github.com/facebookresearch/demucs); it is also effective at isolating vocals even if there is no music
- use `demucs=True` and `vad=True` for music
- set same seed for each transcription (e.g. `random.seed(0)`) for `demucs=True` to produce deterministic outputs
- to enable dynamic quantization for inference on CPU use `--dq true` for CLI or `dq=True` for `stable_whisper.load_model`
- use `encode_video_comparison()` to encode multiple transcripts into one video for synced comparison; see [Encode Comparison](#encode-comparison) 
- use `visualize_suppression()` to visualize the differences between non-VAD and VAD options; see [Visualizing Suppression](#visualizing-suppression)
- [refinement](#refinement) can an effective (but slow) alternative for polishing timestamps if silence suppression isn't effective

### Visualizing Suppression
You can visualize which parts of the audio will likely be suppressed (i.e. marked as silent). 
Requires: [Pillow](https://github.com/python-pillow/Pillow) or [opencv-python](https://github.com/opencv/opencv-python).

#### Without VAD
```python
import stable_whisper
# regions on the waveform colored red are where it will likely be suppressed and marked as silent
# [q_levels]=20 and [k_size]=5 (default)
stable_whisper.visualize_suppression('audio.mp3', 'image.png', q_levels=20, k_size = 5) 
```
![novad](https://user-images.githubusercontent.com/28970749/225825408-aca63dbf-9571-40be-b399-1259d98f93be.png)

#### With [Silero VAD](https://github.com/snakers4/silero-vad)
```python
# [vad_threshold]=0.35 (default)
stable_whisper.visualize_suppression('audio.mp3', 'image.png', vad=True, vad_threshold=0.35)
```
![vad](https://user-images.githubusercontent.com/28970749/225825446-980924a5-7485-41e1-b0d9-c9b069d605f2.png)
Docstring: 
[visualize_suppression()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/stabilization.py#L363-L392)

### Encode Comparison
You can encode videos similar to the ones in the doc for comparing transcriptions of the same audio. 
```python
stable_whisper.encode_video_comparison(
    'audio.mp3', 
    ['audio_sub1.srt', 'audio_sub2.srt'], 
    output_videopath='audio.mp4', 
    labels=['Example 1', 'Example 2']
)
```
Docstring: 
[encode_video_comparison()](https://github.com/jianfch/stable-ts/blob/main/stable_whisper/video_output.py#L29-L73)

#### Multiple Files with CLI 
Transcribe multiple audio files then process the results directly into SRT files.
```commandline
stable-ts audio1.mp3 audio2.mp3 audio3.mp3 -o audio1.srt audio2.srt audio3.srt
```

### Any ASR
You can use most of the features of Stable-ts improve the results of any ASR model/APIs. 
[Just follow this notebook](https://github.com/jianfch/stable-ts/blob/main/examples/non-whisper.ipynb).

## Quick 1.X → 2.X Guide
### What's new in 2.0.0?
- updated to use Whisper's more reliable word-level timestamps method. 
- the more reliable word timestamps allow regrouping all words into segments with more natural boundaries.
- can now suppress silence with [Silero VAD](https://github.com/snakers4/silero-vad) (requires PyTorch 1.12.0+)
- non-VAD silence suppression is also more robust
### Usage changes
- `results_to_sentence_srt(result, 'audio.srt')` → `result.to_srt_vtt('audio.srt', word_level=False)` 
- `results_to_word_srt(result, 'audio.srt')` → `result.to_srt_vtt('output.srt', segment_level=False)`
- `results_to_sentence_word_ass(result, 'audio.srt')` → `result.to_ass('output.ass')`
- there's no need to stabilize segments after inference because they're already stabilized during inference
- `transcribe()` returns a `WhisperResult` object which can be converted to `dict` with `.to_dict()`. e.g `result.to_dict()`

## License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details

## Acknowledgments
Includes slight modification of the original work: [Whisper](https://github.com/openai/whisper)
