# sdl3-mixer

SDL_mixer 3 for sysl — sound and music, mixed, looped, faded and stopped.

```
dependencies {
  sdl3       { git = "github.com/sysl-lang/sdl3",       version = "0.1.2" }
  sdl3-mixer { git = "github.com/sysl-lang/sdl3-mixer", version = "0.1.1" }
}
```

```sysl
import sh.sysl.sdl3.*
import sh.sysl.sdl3_mixer.*

main()
    init(INIT_AUDIO)
    mix_init()

    val mixer = open_mixer().expect("the default device")
    val music = mixer.load_audio("song.ogg").expect("the file decoded")
    val track = mixer.create_track().expect("a voice")

    track.set_audio(music)
    track.play(LOOP_FOREVER)

    delay(5000)

    track.stop(track.ms_to_frames(2000))    // two seconds of fade-out
```

## Installing

```
brew install sdl3_mixer                 # pulls sdl3 with it
sysl run prog.sysl --include-path /opt/homebrew/include --link-path /opt/homebrew/lib
```

The two flags are deliberate — see [`sdl3`](https://github.com/sysl-lang/sdl3)'s README, which also
says why this is a separate package rather than a module inside that one.

## The shape, which is not SDL2_mixer's

SDL_mixer 3 is a redesign, and this is worth reading before anything else:

- a **`Mixer`** owns an audio device and does the mixing;
- an **`Audio`** is a loaded sound — one type covers a two-second effect and an hour of music, with
  `predecode` the only difference, and there is no separate "chunk" and "music" as there was;
- a **`Track`** is a playback voice. Assign an `Audio`, then play, loop, pause, fade and stop *that*.
  A track is reusable, and several may play one `Audio` at once — which is what a game does with an
  effect, and the case a design with one voice per sound gets wrong.

## How a play begins is said at the play

`play(loops, fade_in_frames, start_frame)` — everything about how a play starts goes through the
call that starts it.

That is SDL_mixer's shape rather than a choice, and it is the trap this binding exists to keep you
out of: **`set_loops` alters a track that is already playing and does nothing to a stopped one.**
Its own documentation says so, and the call still answers `true`. A binding that offered only
`set_loops` would let you ask for a loop and get one play, silently. There is a test that asserts
exactly that failure, so the rule is written down where it runs.

`set_loops` is still there, and is useful for what it is actually for: cutting a looping piece of
music short so it stops at the end of the pass it is on.

## Frames, not milliseconds

Everything is in **sample frames**, because that is what a mixer works in and a binding that quietly
converted would be rounding somewhere the caller could not see. `Audio.ms_to_frames` and
`Track.ms_to_frames` convert where it is wanted, against that audio's own sample rate.

The one asymmetry is SDL_mixer's own: `Track.stop` fades over frames while `Mixer.stop_all` fades
over milliseconds. It is kept rather than smoothed over.

`remaining()` counts the **current pass** and knows nothing about looping or fade-outs, so a track
set to loop forever still reports the tail of the pass it is on rather than something infinite.

## Sound a program made itself

`load_raw` takes PCM that is already decoded, which is how a program plays something it generated —
`AudioSpec(AUDIO_F32, 1, 48000)` and a slice of the samples. SDL_mixer copies what it is handed, so
the buffer can go the moment the call returns. The parameter is `[]const u8`, because that is what
the C takes and because both ways of getting there — a `Buf`'s `view`, and a slice of a raw pointer
— answer a read-only view.

**`sine_wave` is a test helper and not a sound effect.** It is raw sine with no envelope, so it stops
wherever in its cycle the length happened to land — ninety milliseconds of 262 Hz is 23.58 cycles and
ends at about half amplitude. That step is broadband, so it is heard as a click at a pitch nothing to
do with the note. Anything meant for a person to listen to wants an envelope that reaches zero at
both ends, which means generating the samples and handing them to `load_raw`.
[`sdl3-demo`](https://github.com/sysl-lang/sdl3-demo) does exactly that, in about fifteen lines.

## Tests

```
sysl test . --include-path /opt/homebrew/include --link-path /opt/homebrew/lib
```

Thirteen tests, and **they need no sound card and no sound file**. SDL's dummy audio driver accepts
a device and consumes what is mixed into it, and SDL_mixer supplies `MIX_CreateSineWaveAudio` — so
the whole path, from opening a device through loading, assigning, seeking, looping and fading, runs
on a build machine with a checkout and nothing else.

## License

ISC — see [LICENSE](LICENSE).
