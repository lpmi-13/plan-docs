# Voice-to-Instrument React Native App Plan

## 1. Product Summary

Build a React Native mobile app that lets a user sing or hum a short melody, converts the recording into a sequence of musical notes, and plays that melody back using a selected instrument such as guitar, trumpet, piano, or tuba.

The first release should prioritize a fast, understandable workflow and convincing note-based playback rather than attempting to reproduce every nuance of the user's voice. The core experience is:

1. Record a short sung or hummed phrase.
2. Detect its pitch, timing, and note boundaries.
3. Preview and optionally correct the detected notes.
4. Choose an instrument.
5. Render and play the instrumental version.
6. Save or share the result.

## 2. Goals and Non-Goals

### Goals

- Make melody conversion approachable for people without music-production experience.
- Support iOS and Android from one React Native codebase.
- Produce recognizable instrumental playback from a monophonic vocal recording.
- Return a useful preview quickly after recording.
- Let users try several instruments without recording the melody again.
- Preserve user privacy and clearly explain when audio leaves the device.
- Design the audio pipeline so better pitch detection and instrument renderers can be introduced later.

### Non-Goals for the MVP

- Converting lyrics or spoken sentences into realistic instrument performances.
- Reliably separating a voice from a full commercial song or noisy backing track.
- Transcribing chords or multiple simultaneous singers.
- Studio-grade timbre transfer that retains every vocal articulation.
- Full digital audio workstation features such as multitrack mixing and automation.
- Generating a backing band or harmonies automatically.

## 3. Target Users and Core Use Cases

### Target Users

- Casual music creators who want to turn an idea into an instrumental phrase.
- Songwriters who need to capture melodies before they are forgotten.
- Music students exploring pitch, rhythm, and orchestration.
- Social-media creators looking for short, shareable audio clips.

### Primary Use Cases

- Hum a melody and hear it on piano.
- Compare the same melody across guitar, brass, and keyboard sounds.
- Correct one or two wrongly detected notes before export.
- Save an idea as audio or MIDI for use in other music software.

## 4. MVP Scope

### Required Features

- Microphone permission onboarding with a short recording tip.
- Record, pause/stop, cancel, and replay a 2–15 second mono clip.
- Input-level meter and recording countdown or elapsed-time display.
- Basic recording validation for silence, clipping, excessive noise, and duration.
- Monophonic pitch detection and note segmentation.
- A simple piano-roll preview showing note pitch, onset, and duration.
- Tap-to-adjust note pitch by semitone and delete an incorrect note.
- Instrument selection with at least:
  - Piano
  - Acoustic guitar
  - Trumpet
  - Tuba
- Immediate preview using cached samples or a local synthesizer.
- Tempo adjustment and octave shift when the selected instrument's range requires it.
- Project save with source recording, detected notes, chosen instrument, and settings.
- Export as an audio file; MIDI export is a high-value stretch goal.
- Share through the platform share sheet.
- Local error reporting and an explicit retry path when conversion fails.

### Post-MVP Features

- More instruments, articulations, reverberation, and effects.
- Trim controls and richer note editing.
- Quantization strength, key snapping, and tempo estimation.
- MIDI and MusicXML export.
- Cloud sync and cross-device projects.
- Background-noise removal and accompaniment separation.
- Polyphonic transcription, harmonization, and generated accompaniment.
- Expressive timbre transfer that maps dynamics, vibrato, and articulation.
- Community sharing, templates, and collaboration.

## 5. User Experience and Main Screens

### 5.1 Welcome and Permissions

- Explain the value in one sentence: “Sing it. Pick an instrument. Hear your idea.”
- Ask for microphone access only when the user starts recording.
- State whether processing is on-device or cloud-based before the first upload.
- Offer recording advice: use headphones, sing one note at a time, and reduce background noise.

### 5.2 Record

- Large, accessible record control reachable with one hand.
- Live waveform or level meter with clipping feedback.
- Maximum-duration indicator and a clear stop action.
- Allow replay and re-record before conversion.

### 5.3 Analyze

- Show a short progress state with meaningful stages such as “Cleaning audio” and “Finding notes.”
- Keep analysis cancellable.
- If the input is unsuitable, explain why and suggest a specific fix rather than showing a generic error.

### 5.4 Edit Melody

- Display the detected melody on a scrollable piano roll.
- Play the original voice and synthesized notes independently for comparison.
- Let the user select a note, move it up or down by semitone, delete it, and undo changes.
- Highlight low-confidence notes so the user knows what to inspect.

### 5.5 Choose Instrument and Preview

- Present instrument cards grouped by family.
- Preview the same note sequence without rerunning transcription.
- Warn before automatic octave transposition and allow the user to override it.
- Keep play, pause, seek, tempo, and octave controls visible.

### 5.6 Save and Export

- Name the project and store it locally.
- Export a standard audio format supported consistently by the target platforms; offer WAV when lossless output matters and AAC/M4A where supported for smaller sharing files.
- Include a share-sheet action and clear export progress.

## 6. Functional Requirements

### Recording

- Capture mono PCM at a consistent sample rate, preferably 44.1 or 48 kHz, then normalize to the analysis model's required rate.
- Route audio correctly around speaker, wired headset, and Bluetooth changes.
- Handle phone calls, app backgrounding, audio interruptions, and revoked permissions.
- Store the original recording until the user deletes its project.

### Analysis

- Reject recordings with no usable voiced section.
- Generate a time-aligned fundamental-frequency curve plus confidence values.
- Convert stable voiced regions into MIDI note number, start time, duration, and velocity.
- Retain the raw pitch contour separately so later versions can support bends and vibrato.
- Complete typical analysis within the performance target defined below.

### Rendering and Playback

- Render from the note representation, not directly from the source audio, for the MVP.
- Respect each instrument's playable range and use sensible transposition rules.
- Avoid audible gaps, clicks, and sample-boundary artifacts.
- Make switching instruments faster than initial analysis by reusing detected notes.
- Allow original and rendered playback without overlapping unexpectedly.

### Projects and Export

- Autosave after analysis and after every edit.
- Version the project schema so it can be migrated later.
- Store source audio, note data, settings, and exported-file references.
- Delete all associated local files when a project is deleted.

## 7. Technical Approach

### 7.1 Recommended Architecture

- **App shell:** React Native with TypeScript.
- **Navigation:** A typed stack for onboarding, recorder, editor, instrument picker, and export screens.
- **State:** Separate transient UI state from persisted project state; use a small predictable store and selectors to limit waveform/editor rerenders.
- **Storage:** SQLite for project metadata and note events, with recordings and renders in the app's private filesystem.
- **Audio engine:** Native iOS and Android modules for low-latency capture, playback, sample scheduling, and offline rendering. Expose a narrow typed interface to JavaScript.
- **Pitch engine:** Begin with a proven monophonic pitch estimator running locally where performance permits. Wrap it behind an interface so a native DSP library, Core ML/TFLite model, or server implementation can be swapped in.
- **Rendering engine:** A sample-based synthesizer or SoundFont-compatible native engine. Ship compact, licensed instrument assets or download optional packs after clear user consent.
- **Observability:** Privacy-conscious crash and performance reporting with no source audio attached by default.

React Native should own screens, project orchestration, and editing interactions. Timing-sensitive audio work should not depend on the JavaScript event loop.

### 7.2 Processing Pipeline

1. Capture mono PCM and persist the original recording.
2. Resample to the detector's expected rate.
3. Apply conservative preprocessing: DC removal, high-pass filtering, and level normalization. Avoid aggressive denoising that changes pitch.
4. Estimate fundamental frequency and confidence for short overlapping frames.
5. Mark unvoiced/low-confidence frames and smooth isolated pitch outliers.
6. Segment the contour into note events using pitch stability, onset energy, and minimum-duration rules.
7. Convert frequency to MIDI pitch with `69 + 12 × log2(frequency / 440)` and retain tuning deviation.
8. Estimate note velocity from vocal energy within a bounded range.
9. Optionally quantize timing lightly, with the unquantized sequence preserved.
10. Apply instrument-range mapping and user-selected tempo/octave settings.
11. Schedule instrument samples for live preview or render them offline for export.

### 7.3 Note Data Model

Each project should contain:

- Project ID, schema version, name, and timestamps.
- Source recording path, format, sample rate, duration, and content hash.
- Analysis version and configuration.
- Optional detected tempo and tuning reference.
- Note events containing ID, MIDI pitch, original cents offset, start time, duration, velocity, confidence, and user-edited flag.
- Selected instrument ID, octave shift, tempo multiplier, effects settings, and render version.
- Export history and local file references.

Keep detected values and user edits distinguishable so reanalysis does not silently overwrite the user's work.

### 7.4 On-Device Versus Cloud Processing

Use on-device processing for the MVP if validation shows acceptable speed and quality on the minimum supported phones. Benefits include offline use, low marginal cost, and stronger privacy. A cloud fallback may improve model quality but introduces upload latency, operational cost, authentication, retention policies, and compliance work.

If cloud processing is introduced:

- Obtain explicit consent before uploading audio.
- Encrypt transport and storage.
- Use short, documented retention periods and automatic deletion.
- Provide a delete request mechanism and an on-device-only mode.
- Send an idempotency key and analysis version with each job.
- Use signed upload/download URLs, job-status polling or push completion, timeouts, and safe retries.
- Never use recordings to train models unless the user separately opts in.

## 8. Pitch Detection and Musical Rules

### Input Assumptions

- One singer or hummer at a time.
- Minimal background music.
- Mostly discrete notes, though short slides and vibrato are expected.
- Typical human vocal range, with the renderer responsible for adapting to instrument ranges.

### Initial Algorithm Options

- Establish a lightweight DSP baseline using YIN or probabilistic YIN.
- Evaluate a neural monophonic pitch model if it materially improves noisy-input or octave-error performance within device constraints.
- Compare algorithms on the same labeled test set rather than selecting by demo quality alone.

### Note Segmentation Heuristics

- Ignore frames below a tunable confidence threshold.
- Merge brief gaps within a sustained note.
- Require a minimum note duration to suppress flutter.
- Start a new note after a stable semitone transition or strong onset.
- Use median pitch within a segment rather than a single frame.
- Preserve slides as metadata even if MVP playback snaps to discrete notes.

### Instrument Range Handling

Maintain metadata for each instrument's comfortable and absolute ranges. When a note falls outside the comfortable range:

1. Prefer octave transposition that preserves melodic intervals.
2. Show the applied shift to the user.
3. If no acceptable mapping exists, flag the note instead of silently clamping it.

Guitar playback should initially use a general plucked-guitar patch; realistic string/fret selection and chord voicing can follow later.

## 9. Instrument Content and Licensing

- Use instrument samples or SoundFonts with licenses that permit commercial mobile redistribution.
- Record the license, attribution requirements, source, version, and checksum for every sound pack.
- Keep third-party notices accessible in the app.
- Validate sample loops, tuning, loudness, attack/release behavior, and complete pitch coverage.
- Budget app size early; consider a small built-in piano and downloadable instrument packs if assets become large.
- Do not scrape or redistribute samples with unclear ownership.

## 10. API and Module Boundaries

Define typed interfaces before choosing final libraries:

- `Recorder`: start, stop, cancel, metering events, interruption events, and result metadata.
- `MelodyAnalyzer`: analyze an audio URI, report progress, cancel, and return versioned note data.
- `InstrumentCatalog`: list installed/downloadable instruments and range/license metadata.
- `MelodyPlayer`: load notes and instrument, play, pause, seek, stop, and emit playback position.
- `OfflineRenderer`: render notes and settings to an output URI with progress and cancellation.
- `ProjectRepository`: create, migrate, load, update, list, and delete projects transactionally.
- `Exporter`: validate output, copy it to a user-accessible destination, and invoke sharing.

Use contract tests around these boundaries so native implementations can evolve without destabilizing the UI.

## 11. Non-Functional Requirements

### Performance Targets

- Recorder controls respond within 100 ms under normal load.
- Analysis of a 10-second recording completes in under 5 seconds on the minimum supported device, with a stretch target near real time.
- Instrument switching begins playback within 500 ms when samples are installed.
- Playback remains free from glitches during normal navigation.
- The editor stays responsive with at least 200 note events.

These are starting targets and should be revised after device profiling.

### Accessibility

- Provide labels, roles, hints, and logical focus order for screen readers.
- Do not encode pitch confidence or recording state by color alone.
- Meet platform contrast and minimum touch-target guidance.
- Support dynamic text without hiding essential controls.
- Add haptic feedback and optional audio cues for recording state.
- Provide list-based note editing as an alternative to precise piano-roll gestures.

### Privacy and Security

- Request only microphone and user-initiated file/share access.
- Keep project files in app-private storage by default.
- Avoid recording filenames or audio content in analytics and logs.
- Redact local paths and user-provided names from crash reports.
- Publish a plain-language privacy policy before release.
- Include in-app deletion and document what uninstalling removes.
- Threat-model cloud upload, signed URLs, downloaded sound packs, and exported files.

### Reliability

- Make recording finalization and project writes atomic where possible.
- Recover incomplete recordings after a crash when safe.
- Validate free disk space before recording, downloading, or rendering.
- Verify downloaded sound-pack hashes before loading them.
- Preserve user edits if rendering or sharing fails.

## 12. Testing Strategy

### Automated Tests

- Unit tests for frequency-to-note conversion, smoothing, segmentation, quantization, range mapping, schema migrations, and reducers/store logic.
- Golden-file tests that run fixed audio fixtures through analysis and compare note/onset metrics within tolerances.
- Native integration tests for recording lifecycle, interruption recovery, route changes, playback scheduling, and offline render output.
- Component tests for permission, error, editing, instrument selection, and export states.
- End-to-end tests for record → analyze → edit → choose instrument → export, using injected fixture audio where simulators cannot provide reliable microphone input.
- File cleanup and privacy tests confirming that deletion removes every associated artifact.

### Audio Quality Dataset

Create a consented, versioned evaluation set covering:

- Different vocal ranges, accents, timbres, vibrato, and singing experience.
- Humming and vowel singing.
- Quiet rooms, common household noise, and several microphone qualities.
- Sustained notes, fast passages, repeated notes, slides, and intentional rests.
- Ground-truth MIDI notes and onset/offset annotations.

Track raw pitch accuracy, octave error rate, voiced/unvoiced F1, note pitch accuracy, onset tolerance, and end-to-end listener preference. Do not rely solely on average pitch error; a melody with octave mistakes can score reasonably while sounding clearly wrong.

### Manual Device Matrix

- Oldest supported and current iPhone models.
- Low-, mid-, and high-tier Android devices from multiple vendors.
- Built-in microphone, wired headset, Bluetooth route, speaker, silent mode, and interrupted sessions.
- Low storage, low battery, airplane mode, denied/revoked permission, and background/foreground transitions.

## 13. Analytics and Success Metrics

Collect event metadata only after appropriate consent and never collect source audio by default.

### Product Funnel

- Permission prompt → recording started.
- Recording started → valid recording completed.
- Recording completed → analysis succeeded.
- Analysis succeeded → instrumental preview played.
- Preview played → project saved or exported.
- Instrument switches per successful project.

### Quality and Operational Metrics

- Analysis and render latency by device class and clip duration.
- Silence/noise rejection and conversion failure rates.
- Crash-free and audio-session-error-free rates.
- Percentage of detected notes manually edited, as a quality proxy.
- Export success and retry rates.
- Sound-pack download size and failure rate.

Define a launch threshold after the prototype baseline; for example, at least 90% analysis completion on supported devices and a strong majority of evaluators rating the melody as recognizable.

## 14. Delivery Plan

### Phase 0: Discovery and Validation (1–2 weeks)

- Interview target users and test the proposed workflow with a clickable prototype.
- Collect or license a small evaluation dataset with ground-truth notes.
- Benchmark two pitch-detection approaches on representative iOS and Android devices.
- Test sample-based playback using one instrument.
- Confirm sample licensing and expected app-size budget.
- Decide the minimum supported OS/device range.

**Exit criteria:** A 10-second clean monophonic melody converts with acceptable accuracy and latency on minimum hardware, and users understand the record/edit/instrument workflow.

### Phase 1: Technical Prototype (2–3 weeks)

- Implement native recording, a fixture-driven analyzer, note representation, and piano playback.
- Build a minimal waveform/piano-roll view.
- Profile CPU, memory, thermal behavior, and playback timing.
- Validate export on both platforms.

**Exit criteria:** An internal build completes the full pipeline on physical iOS and Android devices with no manual file transfer.

### Phase 2: MVP Foundation (3–4 weeks)

- Establish navigation, typed module boundaries, persistence, migrations, and project lifecycle.
- Implement permissions, interruptions, recording validation, and actionable errors.
- Integrate the selected pitch detector and note segmentation.
- Add autosave and recovery behavior.

**Exit criteria:** The core record/analyze/save flow is reliable and covered by unit and integration tests.

### Phase 3: Instruments and Editing (3–4 weeks)

- Add four licensed instrument packs and instrument-range rules.
- Implement piano-roll correction, undo, tempo, and octave controls.
- Add fast instrument switching and offline rendering.
- Complete audio/MIDI export decisions and sharing flow.

**Exit criteria:** Users can correct a melody and render it consistently with every launch instrument.

### Phase 4: Quality, Accessibility, and Beta (2–3 weeks)

- Run the evaluation dataset and tune thresholds without overfitting.
- Complete accessibility review and manual device matrix.
- Add consent-aware analytics, crash reporting, privacy copy, and deletion verification.
- Conduct internal and limited external beta testing.
- Fix release-blocking audio-session, data-loss, and rendering defects.

**Exit criteria:** Quality targets are met, no critical accessibility or privacy issues remain, and store review materials are ready.

### Phase 5: Launch and Follow-Up

- Roll out gradually and monitor conversion, crash, latency, and export metrics.
- Keep a rollback path for analyzer and sound-pack versions.
- Prioritize improvements using note-edit rate, user feedback, and failed-input categories.
- Evaluate cloud analysis or expressive synthesis only after the local MVP baseline is understood.

## 15. Team and Workstreams

A lean team can parallelize the following workstreams:

- **Product/design:** Research, flows, usability, visual design, accessibility, and beta feedback.
- **React Native:** Screens, state, project management, editor interactions, and sharing.
- **Audio/native:** Recording, session routing, playback, offline rendering, and native bridges.
- **Audio ML/DSP:** Pitch estimation, segmentation, dataset curation, evaluation, and tuning.
- **Quality:** Automation, device lab, audio golden tests, release checks, and regression triage.
- **Backend/operations (only if cloud is used):** Uploads, jobs, deletion, monitoring, cost controls, and incident response.

Assign a single owner to the end-to-end audio experience so boundaries between recording, transcription, and rendering do not hide quality problems.

## 16. Key Risks and Mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Noisy audio or expressive singing causes wrong notes | The result sounds unlike the user's idea | Set clear input guidance, expose confidence, allow correction, preserve raw contour, and evaluate diverse fixtures |
| Octave errors in pitch detection | Melody is recognizable but musically incorrect | Add temporal smoothing and vocal-range priors; track octave-error rate separately |
| JavaScript timing causes audio glitches | Poor playback experience | Keep capture, scheduling, and rendering in native real-time-safe code |
| Instrument packs make the app too large | Slow install and higher abandonment | Ship a compact starter set and use verified optional downloads |
| Sample licenses are incompatible | Legal or launch blocker | Maintain an asset ledger and legal review before integration |
| Processing is slow on older devices | Drop-off during conversion | Benchmark early, optimize/resample, show progress, and consider an explicit cloud fallback |
| Audio is uploaded unexpectedly | Loss of trust and compliance exposure | Prefer on-device processing; require explicit, revocable consent for cloud use |
| Automatic transposition surprises users | Incorrect perceived rendition | Show range changes and provide octave controls |
| Device/vendor audio differences | Platform-specific failures | Use a physical-device matrix and instrument audio-session telemetry without content |
| Users expect full timbre transfer | Product disappointment | Describe MVP as melody transcription and instrument playback; demonstrate limitations during onboarding |

## 17. Open Decisions

Resolve these during discovery and the technical prototype:

1. What minimum iOS/Android versions and lowest device class will be supported?
2. Is all analysis required to work offline, or is an opt-in cloud fallback acceptable?
3. Which pitch estimator best meets measured quality, binary-size, license, and latency needs?
4. Which sample engine and file format can be shipped safely on both platforms?
5. Will instrument packs be bundled, downloadable, or a hybrid?
6. Is MIDI export required for launch or shortly afterward?
7. Should timing be left expressive by default, lightly quantized, or user-selectable?
8. What retention and account model, if any, is needed beyond local projects?
9. What objective and listener-evaluation thresholds define acceptable melody quality?
10. Which markets and age groups affect privacy, consent, and store requirements?

## 18. MVP Definition of Done

The MVP is ready for release when:

- A user can record a 2–15 second monophonic vocal melody on supported iOS and Android devices.
- The app detects notes locally or through a clearly disclosed, consented service.
- The user can identify and correct questionable notes.
- The melody can be previewed on piano, acoustic guitar, trumpet, and tuba without reanalysis.
- The user can save a project, reopen it without data loss, export audio, share it, and delete all associated data.
- The pipeline meets agreed latency, stability, recognition-quality, accessibility, and privacy thresholds on the supported device matrix.
- All bundled samples have documented redistribution rights and required attribution.
- Automated tests cover musical rules and project persistence, while physical-device tests cover recording, interruption, playback, and export.
- Monitoring can detect crashes and pipeline failures without collecting users' recordings or melodies by default.
