# gesture-lab

A collection of browser-based creative tools built on [MediaPipe](https://developers.google.com/mediapipe) hand and face tracking. No install, no build step — open in a browser and go.

---

## Projects

### Air Theremin — `index.html`

Control sound with your hands and face.

- **Right hand height** → pitch (C2–C6)
- **Left hand height** → volume
- **Gestures** → waveform shape (open palm = sine, victory = sawtooth, fist = square, thumbs up = triangle)
- **Smile** → snaps pitch to major pentatonic scale

Uses both hand gesture recognition and face landmark tracking.

---

### Hands Theremin — `hands.html`

Same as the Air Theremin but without the face model — faster to load.

- Press **S** to toggle major scale snap

---

### Face Lab — `face.html`

A scaffold for building face-reactive experiences. Tracks 468 face landmarks and 52 blendshape expression scores in real time.

Live expression dashboard shows: smile, blink, jaw open, brow movement, cheek puff.

Add your own code inside `onFaceUpdate(landmarks, blendshapes)`.

---

### Circle → Wave — `circle.html`

An educational visualization for teaching trigonometry and frequency.

Point your index finger and swing it in a circle. The unit circle diagram, sine wave, and cosine wave update in real time — with projection lines connecting circular motion to wave output.

- Faster spin → higher frequency
- Bigger circle → larger amplitude
- Audio pitch tracks rotation speed

---

### Gesture Drum Pad — `drums.html`

Seven hand gestures, seven drum sounds — all synthesized with Web Audio API.

| Gesture | Sound |
|---------|-------|
| ✊ Closed Fist | Kick |
| 🖐 Open Palm | Snare |
| ✌️ Victory | Hi-Hat |
| ☝️ Pointing Up | Open Hat |
| 👍 Thumb Up | Clap |
| 👎 Thumb Down | Low Tom |
| 🤟 ILoveYou | Crash |

Hand height modifies tone — higher hand = brighter/punchier, lower hand = deeper.

---

## Running locally

```bash
python3 -m http.server 8766
```

Then open `http://localhost:8766`.

## Tech

- [MediaPipe Tasks Vision](https://developers.google.com/mediapipe/solutions/vision/overview) — hand landmarks, gesture recognition, face landmarks
- Web Audio API — all sound synthesized in-browser, no samples
- Canvas 2D — all rendering
- Zero dependencies, zero build step
