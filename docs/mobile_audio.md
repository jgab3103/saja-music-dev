mobile_audio

## Mobile Web Audio — Proven BeatMatrix Fix

When developing or debugging JavaScript audio applications for my website, use the following approach, which successfully resolved an iOS Safari playback problem in my BeatMatrix metronome.

### The original problem

The application worked correctly on desktop browsers but produced no sound on iPhone and iPad.

The Web Audio API was functioning, the `AudioContext` reported a running state, and musical events were being scheduled correctly. However, there was no audible output.

Calling `AudioContext.resume()` alone did not resolve the problem.

A diagnostic revealed that playing an HTML5 audio element caused the previously silent Web Audio metronome to become audible.

### The solution that worked

We implemented a **dual-path audio activation mechanism**, triggered synchronously by the user's Play button.

1. Create or reuse a single `AudioContext`.
2. Call `audioContext.resume()` directly inside the user gesture, without awaiting other asynchronous operations first.
3. Create a small, valid PCM WAV file dynamically in JavaScript using an `ArrayBuffer`, `DataView` and `Blob`.
4. The WAV contains a very low-amplitude, nonzero sine wave (effectively inaudible).
5. Create an `HTMLAudioElement` using the generated WAV's object URL.
6. Set the element to loop and call `.play()` directly inside the same user gesture.
7. Keep the HTML audio element playing while the Web Audio application is running.
8. Start normal Web Audio synthesis and scheduling.
9. When stopping playback, pause the HTML audio element but retain the audio context for subsequent playback.

### Reference implementation

```javascript
let audioContext = null;
let unlockElement = null;
let unlockUrl = null;

function activateAudio() {
    const AudioContextClass =
        window.AudioContext || window.webkitAudioContext;

    if (!AudioContextClass) {
        throw new Error("Web Audio is not supported");
    }

    if (!audioContext || audioContext.state === "closed") {
        audioContext = new AudioContextClass();
    }

    // Initiate resume directly inside the user gesture.
    audioContext.resume().catch(console.warn);

    if (!unlockElement) {
        const sampleRate = 22050;
        const sampleCount = sampleRate * 2;

        const buffer = new ArrayBuffer(44 + sampleCount * 2);
        const view = new DataView(buffer);

        function writeString(offset, value) {
            for (let i = 0; i < value.length; i++) {
                view.setUint8(offset + i, value.charCodeAt(i));
            }
        }

        // WAV header: mono, 16-bit PCM.
        writeString(0, "RIFF");
        view.setUint32(4, 36 + sampleCount * 2, true);
        writeString(8, "WAVE");
        writeString(12, "fmt ");

        view.setUint32(16, 16, true);
        view.setUint16(20, 1, true);
        view.setUint16(22, 1, true);
        view.setUint32(24, sampleRate, true);
        view.setUint32(28, sampleRate * 2, true);
        view.setUint16(32, 2, true);
        view.setUint16(34, 16, true);

        writeString(36, "data");
        view.setUint32(40, sampleCount * 2, true);

        // Near-silent but nonzero audio.
        for (let i = 0; i < sampleCount; i++) {
            const sample = Math.round(
                2 * Math.sin(
                    2 * Math.PI * 440 * i / sampleRate
                )
            );

            view.setInt16(44 + i * 2, sample, true);
        }

        unlockUrl = URL.createObjectURL(
            new Blob([buffer], { type: "audio/wav" })
        );

        unlockElement = new Audio(unlockUrl);
        unlockElement.loop = true;
        unlockElement.volume = 1;
    }

    // IMPORTANT: call directly within the user gesture.
    unlockElement.play().catch(console.warn);

    return audioContext;
}
```

### Connecting it to the Play button

```javascript
playButton.addEventListener("click", () => {
    if (isPlaying) {
        stopPlayback();
        unlockElement?.pause();
        return;
    }

    // Both activation operations happen synchronously
    // inside the user's click handler.
    const context = activateAudio();

    startPlayback(context);
});
```

`startPlayback()` should handle the context's asynchronous readiness before scheduling its first events.

### Important implementation requirements

**Do not replace this with a simple `await audioContext.resume()` implementation without testing.** That approach was insufficient in the original iOS Safari tests.

Do not generate separate WAV files for every note, chord or musical exercise. The WAV is used only for audio activation. All actual musical content should continue to use Web Audio synthesis, audio buffers and scheduled playback.

Reuse the same `AudioContext` and HTML audio element across playback sessions. Avoid repeatedly closing and recreating them.

For metronomes, sequencers and other timing-sensitive applications, schedule musical events using `AudioContext.currentTime`, not JavaScript timers alone.

Handle interruptions, browser backgrounding and audio context suspension. If necessary, pause and resynchronise the musical transport after recovery.

### Cross-platform considerations

This solution was verified on iOS Safari in BeatMatrix. It should be compatible with modern Android browsers, but Android behaviour has not yet been independently verified.

The additional HTML audio element is a compatibility workaround, not a requirement of the Web Audio specification or a guarantee of universal compatibility.

### What I want you to do

When building or modifying any of my JavaScript music applications:

- Use this proven activation approach as the starting point for mobile audio compatibility.
- Preserve existing musical functionality and timing accuracy.
- Separate audio activation, synthesis, scheduling and UI rendering.
- Prefer a reusable audio module that can be shared across my exercises.
- Do not remove the dual-path activation workaround unless an alternative has been tested successfully on real iOS devices.

**The objective is reliable, dynamically generated musical audio on desktop, iOS and Android without needing to pre-render every musical event as a WAV file.**