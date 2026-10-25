# Qur'an word-by-word timings

Open, word-level timestamps for complete Qur'an recitations that had none — so apps can
highlight each word as the reciter says it, loop a phrase for memorisation, or jump to a word.

**Licence: CC0 1.0 — public domain.** Use it for anything, no permission or credit needed.

**Website and live demo:** https://its3li.github.io/quran-word-timings/ — pick a surah, press play, watch
the words light up.

> بيانات مفتوحة لتوقيت كل كلمة من كلمات القرآن الكريم في تلاوات كاملة لم يكن لها توقيت كلمة بكلمة:
> لتظليل الكلمة لحظة نطقها، وتكرار المقاطع للحفظ، والانتقال إلى أي كلمة. الترخيص CC0: ملك عام،
> استعملها كما تشاء دون إذن ولا ذكر للمصدر. وإن نفعتك فلا تنسَ من عمل عليها من دعائك.

## What's here

| Reciter | Recording | Surahs | Words | Audio |
| --- | --- | --- | --- | --- |
| محمد اللحيدان — Muhammad Al-Luhaidan | [mp3quran.net](https://mp3quran.net) (`muhammad-luhaidan/r1`) | 114 | 77,429 | 25.3 h |

The audio is **not** included: every file points to the recording's own URL and gives its
SHA-256, so you can check you are timing the same file. Word numbers are quran.com's
(verse key + word position), so the text itself comes from your own Qur'an text source.

```
index.json                              every reciter and surah, audio links, counts
data/<reciter>/quran-com/NNN.json       quran.com / QUL format — drop-in for apps that use it
data/<reciter>/timings/NNN.json         full detail: every word, repeats, confidence, flags
```

### `quran-com/NNN.json` — the format quran.com and QUL use

```json
{"surah": 1, "audio_url": "https://cdn.mp3quran.net/audio/muhammad-luhaidan/r1/001.mp3",
 "verse_timings": [{"verse_key": "1:1", "timestamp_from": 5400, "timestamp_to": 9040,
   "segments": [[1, 5400, 5760], [2, 5800, 6500], [3, 6560, 7500], [4, 7560, 8240]]}, …]}
```

`segments` are `[word position, start ms, end ms]` in time order. These recordings are from
prayer and the reciter sometimes goes back and repeats a phrase: then word positions appear
again (e.g. `[9,…], [10,…], [9,…], [10,…]`), exactly as in quran.com's own data for reciters who
repeat. Highlighting by time therefore follows the recitation as it was actually said.

### `timings/NNN.json` — full detail

```json
{"reciter": {…}, "surah": 67, "audio": {"url": "…", "sha256": "…", "duration": 356.88},
 "words": [{"key": "67:8:9", "ayah": "67:8", "position": 9, "start": 86.98, "end": 87.72, "score": -0.03},
           {"key": "67:8:9", "ayah": "67:8", "position": 9, "repeat": true, "start": 89.36, "end": 90.08, "score": 0},
           {"key": "67:3:6", "ayah": "67:3", "position": 6, "start": 26.42, "end": 26.54, "score": -4.27, "flags": ["aligner unsure"]}, …]}
```

- `key` — `surah:ayah:word` (quran.com numbering); the basmala, when recited, is `basmala:1–4`.
- `start`, `end` — seconds from the start of the audio file. A word's end never passes the next word's start.
- `repeat` — this is the reciter saying the word again.
- `score` — the aligner's confidence (mean log-probability of the word's letters; near 0 = sure).
- `flags` — why a word might deserve a listen (2.9 % of words): `aligner unsure`, `the two AIs
  disagree`, `speech before this word that is not in the text`.

## Using it — highlight words as they are recited

Every file is served free by jsDelivr's CDN, no server needed:
`https://cdn.jsdelivr.net/gh/its3li/quran-word-timings@v1.0.0/data/muhammad-al-luhaidan/quran-com/067.json`

```js
const base = 'https://cdn.jsdelivr.net/gh/its3li/quran-word-timings@v1.0.0/data/muhammad-al-luhaidan';
const t = await (await fetch(`${base}/quran-com/067.json`)).json();
const audio = new Audio(t.audio_url);
const seg = t.verse_timings.flatMap((v) => v.segments.map(([w, a, b]) => ({key: `${v.verse_key}:${w}`, a, b})));
audio.ontimeupdate = () => {
  const ms = audio.currentTime * 1000;
  const now = seg.findLast((s) => s.a <= ms);   // the word being recited
  highlight(now?.key);                           // your UI
};
audio.play();
```

## How it was made

1. **Where each ayah is.** Tarteel's Qur'an model
   ([`tarteel-ai/whisper-base-ar-quran`](https://huggingface.co/tarteel-ai/whisper-base-ar-quran)),
   run locally with whisper.cpp, listens to the whole surah.
2. **Where each letter is.** A Qur'an-trained wav2vec2 model
   ([`rabah2026/wav2vec2-large-xlsr-53-arabic-quran-v_final`](https://huggingface.co/rabah2026/wav2vec2-large-xlsr-53-arabic-quran-v_final),
   4-bit ONNX by [`mostafanewir70`](https://huggingface.co/mostafanewir70/quran-recitation-ctc-onnx))
   gives letter probabilities every 20 ms, and **forced alignment** finds when each letter of the
   **known** mushaf text is said. Nothing is transcribed into the data: it holds times for word
   numbers only, so it can never contain a wrong word or haraka.
3. **Repeats.** The model "corrects" a repeated phrase away when it has just heard it, leaving a
   stretch of loud audio with no letters placed. Such stretches are re-heard on their own, read as
   a path through the text that may jump back, and checked against the audio letter by letter.
4. **Checks.** Words the aligner is unsure of, where the two models disagree, or after speech that
   isn't in the text are flagged rather than hidden.

## How accurate it is

- **Method benchmark** (reciters that already have quran.com word timings — Alafasy, Al-Husary,
  Abdul Basit mujawwad; surahs 1, 67, 78): word starts within 0.2 s of quran.com's for **86 %** of
  words (**93 %** in murattal), counting quran.com's convention that a word's time includes the pause
  before it. Where the two differed by more than 0.25 s, a blind listening test of 101 words chose
  the aligner 90 times and quran.com 8 (3 both fine) — most large differences were errors in the
  reference.
- **This reciter** has no reference to compare with. Al-Mulk was watched in full, and every repeat
  in Al-Baqarah was checked by ear; each mistake found became a rule and a permanent test
  (24 cases, all passing). Across the Qur'an, 3,039 repeated word timings were found automatically.
- **Known limits.** Boundaries inside long, melodic joins (madd, wasl) can be off by ~0.3 s; a
  restart that is only a breathy fragment may be timed as part of the word around it. Flagged words
  are the place to look first.

Corrections are welcome — open an issue with the surah, ayah and what you hear.

## Credits

Audio: [mp3quran.net](https://mp3quran.net). Word numbering: [quran.com](https://quran.com) /
[QUL](https://qul.tarteel.ai). Models: Tarteel AI, rabah2026, mostafanewir70 (all Apache-2.0),
run with [whisper.cpp](https://github.com/ggerganov/whisper.cpp) and
[ONNX Runtime](https://onnxruntime.ai).

## Licence

The timing data is dedicated to the public domain under
[CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/) — see `LICENSE`. The recitations
belong to their owners and are not included.
