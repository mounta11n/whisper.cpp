# Akustisches & Visuelles Feedback Feature Plan

## 🎯 Ziel
Nach Implementierung des Conversation-Mode: Akustisches und visuelles Feedback hinzufügen, um dem Benutzer den aktuellen Zustand klar zu signalisieren.

## 📌 Voraussetzung
- Branch `conversation-mode-main` muss vollständig implementiert und getestet sein
- Neuen Branch von `conversation-mode-main` erstellen (z.B. `conversation-feedback`)

---

## 🔊 Akustisches Feedback

### Mögliche Events für Sound-Feedback:
| Event | Sound-Typ | Beschreibung |
|-------|-----------|--------------|
| Wake-Word erkannt | Kurzer Bestätigungston | z.B. aufsteigender Zweiklang |
| Conversation-Mode aktiviert | Sanfter "Ready"-Sound | Signalisiert: "Ich höre zu" |
| Timeout-Warnung (5s vor Ende) | Dezenter Hinweiston | Optional: kurzer Ping |
| Conversation-Mode beendet | Absteigender Ton | Signalisiert: "Ich warte auf Wake-Word" |
| Fehler/Nicht verstanden | Error-Sound | Optional |

### Implementierungsoptionen:

#### Option A: Externe Sound-Dateien (Empfohlen)
```cpp
// Neue Parameter in whisper_params:
std::string sound_wake     = "";  // Pfad zu WAV-Datei für Wake-Bestätigung
std::string sound_timeout  = "";  // Pfad zu WAV-Datei für Timeout
std::string sound_ready    = "";  // Pfad zu WAV-Datei für "Ready"

// CLI-Argumente:
// --sound-wake FILE    Sound bei Wake-Word-Erkennung
// --sound-timeout FILE Sound bei Conversation-Timeout
// --sound-ready FILE   Sound wenn bereit zuzuhören
```

**Vorteile:**
- Benutzer kann eigene Sounds verwenden
- Keine zusätzlichen Dependencies
- SDL2 Audio bereits verfügbar (für Mikrofon-Capture)

**SDL2 Audio Playback Code-Skeleton:**
```cpp
#include <SDL.h>
#include <SDL_audio.h>

void play_sound_file(const std::string& path) {
    if (path.empty()) return;
    
    SDL_AudioSpec wav_spec;
    Uint32 wav_length;
    Uint8 *wav_buffer;
    
    if (SDL_LoadWAV(path.c_str(), &wav_spec, &wav_buffer, &wav_length) == NULL) {
        fprintf(stderr, "Could not load sound: %s\n", SDL_GetError());
        return;
    }
    
    SDL_AudioDeviceID device = SDL_OpenAudioDevice(NULL, 0, &wav_spec, NULL, 0);
    if (device == 0) {
        fprintf(stderr, "Could not open audio device: %s\n", SDL_GetError());
        SDL_FreeWAV(wav_buffer);
        return;
    }
    
    SDL_QueueAudio(device, wav_buffer, wav_length);
    SDL_PauseAudioDevice(device, 0);
    
    // Wait for playback to finish
    while (SDL_GetQueuedAudioSize(device) > 0) {
        SDL_Delay(10);
    }
    
    SDL_CloseAudioDevice(device);
    SDL_FreeWAV(wav_buffer);
}
```

#### Option B: System-Sounds (macOS spezifisch)
```cpp
// macOS: NSSound oder afplay verwenden
void play_system_sound(const std::string& sound_name) {
    #ifdef __APPLE__
    std::string cmd = "afplay /System/Library/Sounds/" + sound_name + ".aiff &";
    system(cmd.c_str());
    #endif
}

// Beispiel-Sounds auf macOS:
// - Ping.aiff
// - Pop.aiff
// - Tink.aiff
// - Glass.aiff
// - Submarine.aiff
```

#### Option C: Einfacher Beep (Plattformübergreifend)
```cpp
// Minimale Lösung: Terminal-Bell
void beep() {
    printf("\a");
    fflush(stdout);
}
```

---

## 👁️ Visuelles Feedback

### Terminal-basiertes visuelles Feedback:

#### Option A: ANSI Farben (Bereits im Code vorhanden!)
Der Code verwendet bereits ANSI-Codes für fette Schrift:
```cpp
// Zeile 541 in talk-llama.cpp:
printf("%s : the wake-up command is: '%s%s%s'\n", __func__, "\033[1m", wake_cmd.c_str(), "\033[0m");
```

**Erweiterung für Status-Anzeige:**
```cpp
// ANSI Farbcodes
#define ANSI_RESET   "\033[0m"
#define ANSI_BOLD    "\033[1m"
#define ANSI_RED     "\033[31m"
#define ANSI_GREEN   "\033[32m"
#define ANSI_YELLOW  "\033[33m"
#define ANSI_BLUE    "\033[34m"
#define ANSI_CYAN    "\033[36m"

// Status-Anzeige Funktionen
void print_status_waiting() {
    printf("\r[%s●%s WAITING] ", ANSI_YELLOW, ANSI_RESET);
    fflush(stdout);
}

void print_status_active() {
    printf("\r[%s●%s ACTIVE ] ", ANSI_GREEN, ANSI_RESET);
    fflush(stdout);
}

void print_status_listening() {
    printf("\r[%s●%s LISTEN ] ", ANSI_CYAN, ANSI_RESET);
    fflush(stdout);
}

void print_status_thinking() {
    printf("\r[%s●%s THINK  ] ", ANSI_BLUE, ANSI_RESET);
    fflush(stdout);
}
```

#### Option B: Status-Leiste mit Countdown
```cpp
void print_timeout_countdown(int seconds_remaining) {
    printf("\r[ACTIVE: %02ds] ", seconds_remaining);
    fflush(stdout);
}
```

#### Option C: Kompakte Inline-Indikatoren
```cpp
// Symbole für verschiedene Zustände:
// 🎤 - Listening
// 💭 - Thinking
// 🗣️ - Speaking
// 😴 - Waiting for wake word
// ⏳ - Timeout warning

// Oder ASCII-freundlich:
// [MIC] - Listening
// [...]  - Thinking
// [TTS] - Speaking
// [ZZZ] - Waiting
// [!!!] - Timeout warning
```

---

## 🎨 Vorgeschlagene Implementierung (Kombination)

### Neue Parameter:
```cpp
// In whisper_params struct:
bool visual_feedback     = false;  // Aktiviert farbige Status-Anzeige
bool audio_feedback      = false;  // Aktiviert Sound-Feedback
std::string sound_wake   = "";     // Pfad zu Wake-Sound
std::string sound_end    = "";     // Pfad zu End-Sound
int timeout_warning_ms   = 5000;   // Warnung X ms vor Timeout

// CLI-Argumente:
// --visual-feedback       Aktiviert farbige Terminal-Status
// --audio-feedback        Aktiviert Sound-Feedback
// --sound-wake FILE       Custom Wake-Sound
// --sound-end FILE        Custom Timeout-Sound
// --timeout-warning N     Warnung N ms vor Timeout (0 = keine Warnung)
```

### Integration Points im Code:

```cpp
// Nach Wake-Word erkannt (~Zeile 614):
if (visual_feedback) print_status_active();
if (audio_feedback && !sound_wake.empty()) play_sound_file(sound_wake);

// Bei Timeout-Warnung (in Hauptschleife):
if (conv_state == ACTIVE_CONVERSATION && params.timeout_warning_ms > 0) {
    auto remaining = params.conv_timeout_ms - elapsed;
    if (remaining <= params.timeout_warning_ms && remaining > params.timeout_warning_ms - 100) {
        if (visual_feedback) print_timeout_countdown(remaining / 1000);
        // Optional: Warning beep
    }
}

// Bei Timeout (~Zeile 578):
if (visual_feedback) print_status_waiting();
if (audio_feedback && !sound_end.empty()) play_sound_file(sound_end);

// Bei VAD-Erkennung (Sprache erkannt):
if (visual_feedback) print_status_listening();

// Bei LLM-Generierung:
if (visual_feedback) print_status_thinking();
```

---

## 📂 Beispiel Sound-Dateien

Für erste Tests könnten folgende freie Sounds verwendet werden:

| Sound | Beschreibung | Mögliche Quelle |
|-------|--------------|-----------------|
| wake.wav | Kurzer aufsteigender Ton | freesound.org |
| timeout.wav | Sanfter absteigender Ton | freesound.org |
| warning.wav | Dezenter Ping | System-Sounds |

**Wichtig:** Sounds sollten kurz sein (< 500ms), um die Konversation nicht zu stören.

---

## 🔧 Technische Hinweise

### SDL2 Audio Conflict
- **Problem:** SDL2 wird bereits für Mikrofon-Capture verwendet
- **Lösung:** Separate Audio-Device für Playback öffnen, oder
- **Alternative:** Sounds in separatem Thread abspielen

### Thread-Safety
```cpp
// Sound in separatem Thread abspielen (non-blocking):
std::thread sound_thread([path]() {
    play_sound_file(path);
});
sound_thread.detach();
```

### Relevante Dateien
- [`examples/talk-llama/talk-llama.cpp`](examples/talk-llama/talk-llama.cpp) - Hauptdatei
- [`examples/talk-llama/common-sdl.h`](examples/talk-llama/common-sdl.h) - SDL Audio-Capture

---

## ✅ Checkliste für spätere Implementierung

- [ ] SDL2 Audio Playback testen (separate Device)
- [ ] ANSI-Farben auf Windows/Linux/macOS testen
- [ ] Sound-Dateien beschaffen (freie Lizenz)
- [ ] CLI-Parameter implementieren
- [ ] Visual Feedback implementieren
- [ ] Audio Feedback implementieren
- [ ] Thread-Safety sicherstellen
- [ ] Dokumentation aktualisieren

---

## 📅 Geschätzte Komplexität
- Visual Feedback: Einfach (nur ANSI-Codes)
- Audio Feedback: Mittel (SDL2 Audio Playback)
- Timeout-Countdown: Einfach
- Thread-Safety: Mittel

**Empfehlung:** Zuerst visuelles Feedback implementieren, dann Audio.
