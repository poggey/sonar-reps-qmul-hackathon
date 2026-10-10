# Sonar Reps

Your phone already has a sonar. Sonar Reps plays near-ultrasonic tones (18.5 to 19.7 kHz) from the iPhone speaker and listens for the echo off your chest. The phase of that echo follows your chest as it moves, so the phone counts push-ups and reads their depth with no camera, no wearable and no special hardware. Built at the QMUL AI Hackathon (Invisible AI), October 2026.

**Demo video:** [youtube.com/watch?v=PdGd_M7TYp4](https://www.youtube.com/watch?v=PdGd_M7TYp4)

## At a glance

Put your phone on the floor and train. It counts your push-ups and refuses half reps, using sound you cannot hear.

| | Measured on an iPhone 16 Pro Max, judged by the rep judge on takes it was not trained on |
|---|---|
| Full push-ups counted | **39 of 40** |
| Half reps refused | **15 of 15** |
| Phone picked up, slid and shaken | **0 counted** |
| Loud room (noise up to half of full scale) | count unchanged |
| Hand waved over the phone | 8 of 10 counted, **not blocked** |
| Kneeling push-ups | 9 of 10 counted with the depth line, **not blocked**; not yet retested with the judge |

One person, one phone, one afternoon. The full table and its limits are under Evaluation; the judge's figures are from `python judge.py`.

## Why sonar

- **No camera.** Nothing records you, it works in the dark, and there is no framing or lighting to get right.
- **Shaking does not count.** Motion counters are fooled by shaking the phone. Here the phone's motion sensor blocks counting while the phone itself moves. What still gets through is listed under Evaluation.
- **Any phone.** It uses the speaker and microphone every phone already has.

## How it works

```
speaker: 3 tones (18.5, 19.1, 19.7 kHz)  ->  chest  ->  mic
mic -> mix each tone to 0 Hz (I/Q) -> low-pass, null the other tones, 1 kHz
    -> remove the static path (2 Hz high-pass)
    -> phase step per sample from all tones at once (pulse-pair, as in Doppler radar)
    -> path change -> chest movement (lambda/4pi per radian)
    -> rep state machine (depth, return, rest)
```

At 19 kHz the wavelength is 1.8 cm, so one full turn of echo phase is 0.9 cm of chest movement. A push-up is about 25 cm, roughly 28 turns.

## Feasibility, checked

- **Standard phones can do it.** A 2018 study recognised squats and other exercises from the Doppler of a 20 kHz tone on a stock Nexus 5X, with 91% accuracy for squats ([Fu et al., Informatics](https://www.mdpi.com/2227-9709/5/2/24)). It did not count reps or test push-ups.
- **iPhone frequency limit.** iPhone 7 and 8 loopback tests drop sharply above 20 kHz, and their authors used 18.5 kHz ([SonicPACT](https://arxiv.org/pdf/2012.04770)). That is why our tones stay below 20 kHz.
- **Millimetre tracking from phase.** LLAP tracked fingers to millimetres with phone speakers and mics ([LLAP code](https://github.com/Samsonsjarkal/LLAP)).
- **Audibility.** 18.5 kHz is inaudible to most adults, but some young people hear a faint whine. Keep the volume moderate.

## Run it

```sh
pip install numpy scipy sounddevice soundfile matplotlib
python simulate.py                       # synthetic tests, no hardware
python probe_live.py --check             # can this laptop do 19 kHz?
python probe_live.py                     # live tracking on the laptop
python phone_receiver.py                 # live tracking from the iPhone
python analyze_wav.py take.wav --plot take.png
python takes.py                          # replay the real recordings in takes/
python judge.py                          # train the rep judge on the labelled takes, test it on unseen ones
python presentation/make_charts.py       # redraw the charts in presentation/img
```

iPhone: `cd ios && xcodegen generate`, open the project and run on a phone. The Mac's IP is a developer setting: open Settings (the gear), tap Version five times, and enter it under Developer (default 172.20.10.11). Then press Start set. The Mac sends its verdict back and the phone shows the count. If the university Wi-Fi blocks the stream, turn on the iPhone's Personal Hotspot and join it from the Mac.

## The phone app

A native SwiftUI app in `ios/`, built around one rule from the sonar: once Start is pressed, the phone lies under your chest and must not be touched. Everything during a set reads at a glance from 30 to 60 cm away.

| Home | During a set | NO REP | Phone moved | After the set |
|---|---|---|---|---|
| ![Home](docs/app/home-light.png) | ![Set](docs/app/set-light.png) | ![NO REP](docs/app/no-rep.png) | ![Phone moved](docs/app/phone-moved.png) | ![Result](docs/app/result-light.png) |

These are rendered from the app's SwiftUI code on a Mac, playing the built-in demo set (target 15 cm in the demo; live, the Mac sends the judge's line). Dark theme: `docs/app/set-dark.png`, `docs/app/result-dark.png`. Also `docs/app/how-it-works.png`, `docs/app/settings.png` and the icon, `docs/app/icon.png`.

- **Home.** Your personal best (reps, when, deepest and average depth, time) and your recent sets. Saved on the phone only.
- **During a set.** A huge count, a depth column with the target line, a bar for every movement, and the trace of your chest over the last 12 seconds, so you can see what the sonar sees. Stop and Restart are large.
- **NO REP.** The whole screen turns red with the reason, for example "Too shallow. You reached 9 cm of 15 cm."
- **Phone moved.** An amber banner, "Phone moved. Keep it still.", and nothing counts until it is still. A similar warning shows when the tones are too quiet to count.
- **After the set.** Reps, NO REPs, time, deepest and average depth, depth per movement, a badge for a new personal best, and one line of advice worked out from the judged reps.
- **Settings.** Light or dark theme, how it works, clear history. Developer mode (tap Version five times) holds the Mac address, the raw recording and connection details, out of a normal user's way.

Rules the design follows, because breaking them breaks the counting:

- **No sound and no vibration.** The speaker is the sonar, and a vibration moves the phone, which blocks counting.
- **No touch during a set.** Start, then hands off.
- **Readable from the floor.** Strong contrast, one accent colour, red only for NO REP and amber only for warnings.
- **Accessible.** VoiceOver reads the count, depth and each rep; Reduce Motion is respected; NO REP is marked by more than colour.

## Status

- Synthetic tests pass: steady, fast (0.7 s) and noisy sets, a shallow rep flagged as no rep, an empty floor counting zero.
- Laptop test on real hardware: all 10 passes of a book over a MacBook Air were detected.
- iPhone 16 Pro Max, face-up on the floor, live to the laptop screen and back to the phone: see Evaluation.
- Today the phone is the speaker and microphone and the laptop does the processing over a local hotspot.
- Every test, including the failed ones, is in `RESULTS.md`.

## Evaluation

Labelled sets recorded on an iPhone 16 Pro Max (10 Oct, 11:22 and 13:19 to 13:27), one person, one room. Each set is judged by the rep judge trained on the other sets only, so it has not seen the set it is judging (`python judge.py`).

| Set | Should count | Counted | |
|---|---|---|---|
| 10 full push-ups (11:22) | 10 | 10 | getting into position refused |
| 10 full push-ups, normal pace | 10 | 10 | |
| 10 full push-ups, fast (0.9 s each) | 10 | 10 | |
| 5 deep push-ups, chest touching down | 5 | 5 | |
| 5 full and 5 half, alternating | 5 | 4 | all 5 half reps refused; 1 full rep refused |
| 10 half reps | 0 | 0 | blocked |
| Phone picked up, slid and shaken | 0 | 0 | blocked; live, the motion sensor also stopped all counting |
| Empty floor, people walking nearby, 30 s | 0 | 1 | one false rep in the first 5 s, none after |
| Hand waved over the phone 10 times | 0 | 8 | **not blocked** |
| 10 kneeling push-ups | 0 | 9 | **not blocked** |

In short: 39 of 40 full push-ups counted, 15 of 15 half reps refused, phone shaking blocked. A hand moved up and down over the phone counts as a rep, and so does a kneeling push-up: the sonar sees one surface moving, not whose it is or where the knees are.

Limits of this table: one person, one phone, one afternoon. Two earlier sets whose true counts were never confirmed read shallower and are not in it.

### The rep judge

`judge.py` finds every down and up, measures it on the way down and on the way back up, and a small trained model (logistic regression, plain numpy, `judge.json`) says full rep or half rep. It judges on two numbers: the larger of the two strokes, and how steadily the echo's phase turned. `phone_receiver.py` uses it by default.

Trained on 55 labelled movements (40 full, 15 half). Tested on sets it had not seen:

| | Full and half reps judged right |
|---|---|
| Rep judge | 54 of 55 |
| One fixed depth line, fitted the same way | 53 of 55 |

Why both strokes: the way down sometimes reads short (one full rep read 8.6 cm down and 16.2 cm back up), and after a pause at the bottom the way up does. The larger of the two is usually right.

What to hold against these figures:

- The two inputs were chosen by scoring feature sets on this same test, so 54 of 55 is somewhat optimistic. The other sets scored 52 to 54.
- A first version with seven inputs and a third verdict, "not a push-up", got 21 of 72: a hand wave gives the same numbers as a push-up, so it cannot be learned from them. Hand waves are tested but not trained on. Telling a hand from a chest needs the distance to the moving surface, which steady tones do not give.
- 55 movements from one person is a small set.

**Depth.** In the deep set the chest went from arms locked (about 30 cm up by Addy's estimate) down to touching, and the sonar read 17.3 to 24.2 cm. So depth is in the right range and probably reads short. It is good enough to sort full from half reps, not to quote in centimetres.

**Volume matters.** With the phone volume turned down the tones were 10 times weaker and a set of 10 read as noise. The receiver now refuses to count and says so, on the laptop and on the phone, when the tones are too quiet.

**The app.** The iPhone screen (SwiftUI, `ios/UI/`) shows the count, a depth column, NO REP, the phone-moved and too-quiet warnings, a result screen with a personal best, and one line of advice after each set. That line is plain rules on the judge's verdicts, not a language model.

## Presenting it

A slide outline for the 3 minute video or a deck. The finished video: [youtube.com/watch?v=PdGd_M7TYp4](https://www.youtube.com/watch?v=PdGd_M7TYp4). Charts and a brief for the slides are in `presentation/`. Every number below is from this README or `RESULTS.md`; nothing planned is said as done.

1. **Hook.** "There is no camera on. This phone is counting your push-ups with sound you cannot hear." Picture: `docs/app/set-light.png` next to the phone on the floor.
2. **The problem.** Camera counters need you to film yourself. Motion counters count a shake as a rep. Both make you manage a screen while you train.
3. **The idea.** Every phone already has a sonar: a speaker and a microphone. Three tones at 18.5, 19.1 and 19.7 kHz, the echo off your chest, and its phase gives movement. At 19 kHz one turn of phase is 0.9 cm of chest movement; a push-up is about 28 turns. Picture: the pipeline under How it works.
4. **Live demo.** Five full reps counting up, one half rep flashing NO REP, the phone picked up and "Phone moved". Pictures: `set-light.png`, `no-rep.png`, `phone-moved.png`.
5. **Does it work.** 39 of 40 full push-ups counted, 15 of 15 half reps refused, shaking the phone blocked, a loud room made no difference. Picture: the At a glance table.
6. **Where the AI is, said plainly.** The tracking is physics. The verdict on each movement is a small trained model, the rep judge, and it had to earn its place: our first model lost to a simple depth line (51 against 53 of 55). Measuring the way back up as well as the way down fixed that: on sets it had never seen it gets 54 of 55 against the line's 53. It is a few dozen numbers, runs with no cloud, and could run on the phone. Picture: the judge table under Evaluation.
7. **What still fools it.** A hand waved over the phone still counts (8 of 10 with the judge), and so do kneeling push-ups (9 of 10 with the depth line): the sonar sees a surface moving, not whose it is. Telling them apart needs the distance to that surface, which steady tones do not give; short chirps would.
8. **The app.** Personal best and history kept on the phone, NO REP and phone-moved verdicts readable from the floor, light and dark themes. Picture: the row of screens under The phone app.
9. **Why it matters.** No camera, so no image of you ever exists; the test recordings kept in this repo hold only the band above 17.5 kHz, so no speech. It needs only a speaker and a microphone (tested so far on one iPhone and one MacBook). Uses: training at home, fitness tests, challenges between friends, places where cameras are not welcome.
10. **Next.** Move the tracker onto the phone so it works alone (today a laptop on the same hotspot does the processing); chirps for distance, to tell a chest from a hand; more movements such as squats.

Honest limits to say out loud: one person, one phone, one afternoon; depth is good enough to sort full from half reps, not to quote in centimetres; with the phone volume turned down it fails, so it now refuses to count and says why.

## Team

Padraig, Ximo, Addy and George.

## Prior work

Reuses ideas, not code, from Addy's earlier PressPass project (camera push-up counter): the rep state machine design and the replay evaluation approach.
