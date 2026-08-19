# Mimics, Muscles, and Me (MMM)

Paired facial surface electromyography (sEMG) and frontal video of a **single participant** performing standardized facial movements and emotional expressions.
Recorded at the Department of Otorhinolaryngology, Jena University Hospital / Friedrich Schiller University Jena.

MMM is the single-subject companion to the [Mimics and Muscles (MaM)](https://github.com/cvjena) dataset.
It follows the same protocol, the same processing chain and the same file layout, but the participant is the author, who consented to a fully open release.
There is no data-use agreement and no consent table: the recordings may be used, shown and published freely, they just may not be redistributed (see `LICENSE.md`).

Every clip is cut to a single task, and the sEMG is resampled to the video frame rate, so signal and image are frame aligned without further work.
The release also contains electrode-removed videos, uncut variants of every clip, and per-frame 3D morphable model (3DMM) parameters from seven monocular 3D face reconstruction methods.

## Summary

| Item | Value |
| --- | --- |
| Participants | 1 (`pid` 99) |
| Sessions | 4 (`1N`, `1S`, `2N`, `2S`) |
| Clips (`MMM.csv` rows) | 95 |
| Clips with sEMG | 46 (about 10 min) |
| Clips with electrode-removed video and 3DMM | 46 |
| Size | about 360 MB |

## Recording design

The participant was recorded in two blocks, each once with sEMG electrodes attached and once without.
The last character of the session label gives the session type.
`S` means electrodes were attached and are visible in the video.
`N` means no electrodes, video only, and no sEMG.
The `N` sessions provide clean-appearance reference video of the same tasks.

There are two task paradigms, given by `mode`:

- `schaede` covers standardized movement exercises, one clip each per session: `Face-At-Rest`, `Forehead`, `Eye-Gentle`, `Eye-Tight`, `Nose-Pucker`, `Smile-Closed`, `Smile-Open`, `Lip-Pucker`, `Cheeks`, `Snarl`, `Depress-Lip`. The `N` sessions carry a twelfth task, `Smile-Natural`. Clips run about 22 s at the median (650 frames), from 12 s to 28 s.
- `emotion` covers the six basic emotions (`angry`, `disgusted`, `fearful`, `happy`, `sad`, `surprised`) in randomized imitations of about 6 s (172 frames). Block 2 has the full set of 24 per session; block 1 has a single clip, because the emotion protocol was not completed there.

| Session | Clips | with sEMG | emotion | schaede |
| --- | --- | --- | --- | --- |
| `99_1N` | 13 | 0 | 1 | 12 |
| `99_1S` | 12 | 12 | 1 | 11 |
| `99_2N` | 36 | 0 | 24 | 12 |
| `99_2S` | 34 | 34 | 23 | 11 |

Video and sEMG ran on independent clocks.
Unlike MaM, this recording has no usable hardware trigger LED in frame, so alignment is done by consensus cross-correlation: the sEMG envelopes of the most active channels are correlated against MediaPipe blendshape trajectories extracted from the electrode-removed video, and the dominant lag cluster wins.
Where no cluster forms, the sEMG trigger channel provides a fallback lag.
Per-clip correlation grades, lags and cut windows are kept outside the release in `10_emg_sync/sync_stats.csv`; of the 46 synced clips, 39 reach Grade 1, 2 Grade 2, 2 Grade 3, and 3 fell back to the trigger.

## File naming

Each clip has one identifier, used as the filename stem everywhere and as `index` in `MMM.csv`:

```text
{pid}_{rec}_{mode}_{task_seq}_{task}
 │     │     │      │          └─ task label
 │     │     │      └─ zero-padded index of the task within the session
 │     │     └─ emotion | schaede
 │     └─ session label: block number 1 or 2, plus S (sensors) or N (none)
 └─ participant id, always 99
```

`99_2S_emotion_12_happy` is block 2 with sensors, the 13th imitation of that session, target *happy*.

```text
{folder}/{pid}_{rec}/{identifier}.{ext}
video_noelec_params/{method}/{pid}_{rec}/{identifier}_params.csv.gz
```

`{pid}_{rec}`, for example `99_2S`, is the session key. The `source` column in `MMM.csv` adds the mode, giving `99_2S_emotion`.

## Directory contents

| Directory | Files | Size | Contents |
| --- | --- | --- | --- |
| `emg/` | 46 | 7 MB | Per-clip sEMG CSV, one row per video frame, 22 channels |
| `video/` | 95 | 90 MB | Frontal video, cut to the synced window |
| `video_no-cut/` | 96 | 96 MB | Same clips before the sync cut, so with the lead-in and lead-out frames |
| `video_noelec/` | 46 | 21 MB | Electrode-removed version of `video/`, synthesized via MC-CycleGAN |
| `video_noelec_params/` | 322 | 147 MB | 3DMM parameters fitted to `video_noelec/`, one subdirectory per method |
| `MMM.csv` | | 20 KB | Clip index, one row per clip |
| `MMM_3DMM.csv` | | 30 KB | Clip index restricted to clips that have 3DMM parameters |

All videos are 286×286, 30 fps, yuv420p, H.264 (`libx264`, `crf 23`), in an `.mp4` container.
`video/`, `video_noelec/` and the 3DMM parameter files of a clip all have exactly the same frame count; only `video_no-cut/` is longer, by 0 to 98 frames.

## sEMG

`emg/{pid}_{rec}/{identifier}.csv` holds an unnamed integer frame index starting at 0, then 22 float columns, one row per video frame at 30 Hz.
The channels are 11 muscles on two sides (`_l` and `_r`), named from the recording protocol and partly German. Column order is the same in every file: alphabetical, with `_l` before `_r`.

```text
Corrugator Supercilii            Masseter
Depressor Ang. Oris              Medialer Frontalis
Depressor Supercilii Procerus    Mentalis
Lateraler Frontalis              Orbicularis Oculi
Levator Labii Superioris         Orbicularis Oris
                                 Zygomaticus
```

The raw recording was 4096 Hz monopolar and was converted to a bipolar Fridlund montage. Each clip then went through per-channel mean removal with NaNs zeroed, bipolar pairing of adjacent electrodes per side, a 50 Hz notch filter (IIR, Q = 2), a 10 to 500 Hz FIR bandpass (129 taps, Hamming, zero phase), a sliding-window RMS, and a resample to 30 Hz.

The RMS window is 2048 samples, about 500 ms at 4096 Hz, and it dominates the chain. The result is an envelope, not raw sEMG, and short bursts come out broadened.

Values are RMS envelope amplitude in the recorder's native export units, on a microvolt scale, with a median around 7 and a maximum around 1800. No normalization has been applied. Absolute amplitude depends on electrode placement and skin impedance, so normalize per session before comparing across sessions. The Fourier resample to 30 Hz can ring slightly negative at clip edges; clamp at zero if a strictly non-negative envelope is needed.

sEMG row *i* matches video frame *i* of `video/` and of `video_noelec/`, aligned at the clip start. In this release the lengths match exactly for all 46 clips.

## `MMM.csv`

This is the authoritative inventory. Iterate over it; the directory tree contains files that no row references. Paths are relative to the dataset root.

| Column | Description |
| --- | --- |
| `source` | `{pid}_{rec}_{mode}` |
| `index` | Clip identifier, unique |
| `pid` | Participant id. Read it as a string: it is zero padded, and pandas will silently convert it to an int |
| `rec` | Session label, `1N`, `1S`, `2N` or `2S` |
| `mode` | `emotion` or `schaede` |
| `task` | Task label |
| `type` | `sensor` if `rec` ends in `S`, `normal` if it ends in `N` |
| `sync_grade` | `0` if a synced sEMG segment exists, `-1` otherwise. This is a presence flag, not the correlation grade; the correlation grades live in `10_emg_sync/sync_stats.csv` |
| `use` | `yes` if a usable synced sEMG segment exists |
| `video` | Cut electrode video. Present for every row |
| `video_no-cut` | Uncut variant of `video`. Present for every row |
| `emg` | sEMG CSV. Empty for every `N` session |
| `video_noelec` | Electrode-removed video. Empty for every `N` session |

The `use` column reports whether a paired sEMG segment exists, so it reads `no` for all 49 electrode-free `N` clips even though their video is fine.
It is not a quality flag.
Filter on whichever condition you actually need:

```python
df = pd.read_csv("MMM.csv", dtype={"pid": str})

paired = df[df.emg.notna()]                        # EMG + video
clean  = df[df.type == "normal"]                   # electrode-free appearance video
uncut  = df["video_no-cut"]                        # untrimmed clips, EMG does not apply
```

## 3DMM parameters

`MMM_3DMM.csv` indexes the 46 clips that have an electrode-removed video, with one path column per method plus `video_noelec` and `emg` so the parameters pair directly with the signal.
It exists as a separate file because its row set is a subset of the one in `MMM.csv`.

The fits use `video_noelec/` rather than the raw electrode video, because the electrode array corrupts reconstruction.
Each file is a gzipped CSV with one row per video frame and no index column, so row *i* lines up with video frame *i* and with sEMG row *i*.

| Method | Columns | Groups |
| --- | --- | --- |
| `DECA` | 236 | `idt`×100, `exp`×50, `pose`×6, `cam`×3, `light`×27, `tex`×50 |
| `EMOCAv2` | 236 | `idt`×100, `exp`×50, `pose`×6, `cam`×3, `light`×27, `tex`×50 |
| `Deep3DFace` | 257 | `idt`×80, `exp`×64, `tex`×80, `angle_{x,y,z}`, `gamma`×27, `t_{x,y,z}` |
| `EIFER` | 361 | `idt`×300, `exp`×50, `eyelid`×2, `jaw`×3, `pose`×3, `cam`×3 |
| `SMIRK` | 361 | `idt`×300, `exp`×50, `eyelid`×2, `jaw`×3, `pose`×3, `cam`×3 |
| `FOCUS` | 531 | `idt`×199, `exp`×100, `tex`×199, `cam`×6, `sh`×27 |
| `MediaPipe` | 52 | Named ARKit-style blendshape coefficients such as `jawOpen` and `mouthSmileLeft` |

All seven methods cover all 46 clips.

## Known gaps

The data itself has these holes; the index is correct about them.

- The `N` sessions use different labels than the `S` sessions for three of the same movements: `-Smile-Closed`, `-Smile-Open` and `Inflate-Cheeks` instead of `Smile-Closed`, `Smile-Open` and `Cheeks`. The leading dashes are a naming artefact of the recording protocol. Match on `task_seq` rather than `task` when pairing `N` and `S` clips.
- The `N` sessions have a twelfth `schaede` task, `Smile-Natural`, that the `S` sessions do not.
- Block 1 has only one `emotion` clip per session, `00_sad`. The rest of that protocol was not recorded.
- `video_no-cut/99_2S/99_2S_emotion_23_angry.mp4` has no counterpart in `video/`, because that clip failed to sync. It is on disk but no row references it.
- One participant only, so nothing here supports claims about between-subject variability.

## Citation

Use of this dataset requires citing the three papers below.
The MC-CycleGAN paper covers the electrode-removed videos in `video_noelec/`, which every 3DMM parameter set is fitted to.

### EIFER

```bibtex
@inproceedings{buechner2025electromyography,
 doi = {10.1109/CVPR52734.2025.00029},
 year = {2025},
 booktitle = {IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)},
 title = {Electromyography-Informed Facial Expression Reconstruction for Physiological-Based Synthesis and Analysis},
 author = {Tim Büchner and  Christoph Anders and Orlando Guntinas-Lichius and Joachim Denzler},
}
```

### MC-CycleGAN

```bibtex
@inproceedings{buechner2023improved,
 doi = {10.1007/978-3-031-45382-3_22},
 pages = {262-274},
 year = {2023},
 booktitle = {Advanced Concepts for Intelligent Vision Systems (Acivs)},
 author = {Tim Büchner and Orlando Guntinas-Lichius and Joachim Denzler},
 title = {Improved Obstructed Facial Feature Reconstruction for Emotion Recognition with Minimal Change CycleGANs},
}
```

### EMG

```bibtex
@article{guntinas2023high,
  title={High-resolution surface electromyographic activities of facial muscles during the six basic emotional expressions in healthy adults: a prospective observational study},
  author={Guntinas-Lichius, Orlando and Trentzsch, Vanessa and Mueller, Nadiya and Heinrich, Martin and Kuttenreich, Anna-Maria and Dobel, Christian and Volk, Gerd Fabian and Gra{\ss}me, Roland and Anders, Christoph},
  journal={Scientific reports},
  volume={13},
  number={1},
  pages={19214},
  year={2023},
  publisher={Nature Publishing Group UK London}
}
```

## License

See `LICENSE.md`, or `MMM_License.pdf` for the same terms typeset.
Free to share and adapt for any purpose, excluding commercial use, with attribution (CC BY-NC 4.0).
Redistribution of the dataset, in whole or in part, is not permitted; point people at the original download instead.
