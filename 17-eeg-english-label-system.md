# EEG-to-English Label System

A brain–computer-interface (BCI) integration. A consumer EEG headset feeds raw brainwave readings over a serial port; the server maps each reading to an English word + a sound + an EXP reward and pushes them back to the client. The result is a real-time biofeedback layer on top of the normal Mir2 gameplay loop.

## What it does

Once per ~5 seconds (per connected headset):
1. Server reads the latest raw EEG value from the player's COM port.
2. Looks the value up in `Envir/EEGMapping.csv` — a 4-column CSV mapping raw value → (EXP, English word, sound id).
3. Awards the EXP to the player.
4. Sends the word + sound to the client.
5. Client renders the word in big white text above the main HUD and plays `./sounds/<id>.mp3`.

Functionally, the system turns brain activity into an in-game stimulus stream: focus → high EXP rolls + positive words; distraction → low EXP + ambient words.

## Where to find it

| Component | File |
|---|---|
| Server serial reader (one per player) | `Server\MirEnvir\EEGSerialManager.cs` |
| Server CSV mapper | `Server\MirEnvir\EEGMapper.cs` |
| CSV data file | `Build\Server\Debug\Envir\EEGMapping.csv` |
| Server tick that drives it | `Server\MirObjects\PlayerObject.cs` — search for `EEGSerial.Process()` |
| Client display dialog | `Client\MirScenes\Dialogs\EEGStimulusDialog.cs` |
| Client-side wrapper | `Client\EEG\EEGManager.cs` |
| Packet | `S.EEGStimulus { string Word; int SoundId; }` |
| Trigger hooks | `TriggerEvent.EEGConnected / EEGDisconnected / EEGValueChanged`<br>`ConditionType.EEGConnected / EEGSignalQuality / EEGAttention / EEGMeditation / …` (enum values 161-170) |

## Hardware

Any **TGAM-protocol** EEG headset (NeuroSky MindWave, MindWave Mobile via the bundled USB dongle, etc.). The serial reader is hardcoded to **57600 baud, 8-N-1** — the TGAM default. Other protocols (Muse, Emotiv, OpenBCI Cyton/Ganglion) are NOT supported by the shipped reader; you'd need a different parser.

You only need:
- The headset.
- A USB-Bluetooth dongle (or built-in BT).
- A COM port number assigned to the headset by Windows.

## EEGMapping.csv format

Plain CSV, one row per raw-value bucket. Comments are lines starting with `#`. Blank lines are ignored.

```
# A,           B,    C,            D
# rawEEG,      EXP,  Word,         SoundId
-3000,         100,  whats,        1
-2999,         101,  acutely,      2
-2998,         102,  tactic,       3
…
 2999,         930,  brilliant,    1995
 3000,         931,  enlightened,  1996
```

| Column | Type | Meaning |
|---|---|---|
| A — Raw EEG | int (negative OK) | The raw integer value the headset produced. Direct exact-match lookup. |
| B — EXP | uint | Experience points awarded to the player when this row hits. `0` = no EXP. |
| C — Word | string ≤20 chars | English word displayed above the HUD for ~3 seconds. |
| D — SoundId | int | Looks up `./sounds/<id>.mp3`. `0` = silent. |

### Lookup rules

```
GetRow(eegValue):
  if exact key exists in the table → return that row
  else                              → fallback bucket = abs(eegValue) % rowCount
                                       return _sortedKeys[fallback bucket]
```

The fallback path guarantees **every possible int value resolves to some row** — there are no "no match" gaps. As a side effect, values outside the table's range still produce meaningful output by wrapping into the row index.

### CSV authoring tips

- **Keep the table dense.** The fallback wraps `abs(value) % count` — gaps don't hurt, but the resolution of mapped words degrades.
- **Pair words to EXP intent.** Don't use "anxious" with high EXP — the player will form a Pavlovian association you don't want.
- **Sound IDs are MP3 files in `./sounds/`.** Pre-bake them. The client uses NAudio to play asynchronously.
- **20-char cap on words.** Anything longer is truncated server-side before send.
- **Reload requires a server restart.** The CSV is read once on `Envir.Start` (around `Envir.cs:2047`). To hot-reload, expose `EEGMapper.Load(...)` via a GM command — `~5 LOC` to add `@EEGRELOAD`.

## How players connect a headset

There's no UI walk-through in the shipped editor — connection is per-`PlayerObject` and gated by the server. Typical flow:

1. Player pairs the headset over Bluetooth → Windows assigns a `COMn` port.
2. Player tells the server which port. Most servers expose this via a GM command or NPC dialog:
   ```csharp
   case "EEG":
       if (parts.Length >= 2 && int.TryParse(parts[1], out int port))
           ReceiveChat(EEGSerial.Connect(port), ChatType.System);
       else
           ReceiveChat(EEGSerial.IsConnected ? "EEG connected." : "Usage: @EEG <comPort>", ChatType.System);
       return;
   ```
3. Server opens the COM port at 57600/8-N-1, spawns the reader thread.
4. Within ~5 seconds, the player should start seeing words flash above their HUD.

If `Connect` fails it returns the underlying exception message verbatim — useful diagnostic.

## Server-side tick

In `PlayerObject.Process()`:

```csharp
int? eegValue = EEGSerial.Process();   // returns null until 5s window elapsed
if (eegValue.HasValue)
{
    LatestEEGRaw = eegValue.Value;
    var row = Envir.Main.EEGMapper.GetRow(eegValue.Value);
    if (row.Exp > 0) GainExp(row.Exp);
    Enqueue(new S.EEGStimulus { Word = row.Word ?? "", SoundId = row.SoundId });
    Envir.Main.Triggers.Raise(TriggerEvent.EEGValueChanged,
        TriggerContext.ForPlayer(TriggerEvent.EEGValueChanged, this));
}
```

The trigger raise lets operators wire `TriggerEvent.EEGValueChanged` to any effect chain — e.g., "if EEGAttention > 80 sustained for 30s → grant a buff."

## Client display

The dialog (`EEGStimulusDialog`):
- 800×70 panel auto-positioned just above `MainDialog`.
- Renders the word at 35 pt bold white with a black outline (readable against any background).
- Auto-hides after 3 seconds.
- Plays `./sounds/<id>.mp3` via NAudio if `SoundId > 0`.

Falls back gracefully if the MP3 is missing (no crash, just silence).

## Trigger system hooks

Available `ConditionType` and `TriggerEvent` values for EEG-driven gameplay:

```
ConditionType:
  EEGConnected (161)            EEGNotConnected (162)
  EEGSignalQuality (163)        EEGRawWave (164)
  EEGAttention (165)            EEGMeditation (166)
  EEGBandPower (167)            EEGFlowState (168)
  EEGAttentionSustained (169)   EEGMeditationSustained (170)

TriggerEvent:
  EEGConnected (46)             EEGDisconnected (47)
  EEGValueChanged (48)
```

Example: trigger a screen flash when the player enters a sustained "flow state."

```json
{
  "Index": 5001,
  "Name": "FlowStateReward",
  "Events": [ "EEGValueChanged" ],
  "RootConditionGroup": {
    "Operator": "AND",
    "Conditions": [
      { "Type": "EEGFlowState", "Params": [ "70", "60" ] }
    ]
  },
  "Effects": [
    { "Type": "AddBuff", "Params": [ "MagicBooster", "60000", "0" ] },
    { "Type": "SendHint", "Params": [ "Flow state — magic boosted!" ] }
  ]
}
```

(The `EEGFlowState` predicate's exact param shape is documented in `Triggers\TriggerCondition.cs`. The 70/60 above means "Attention ≥70 AND Meditation ≥60.")

## Common pitfalls

- **Wrong COM port.** Windows can renumber ports after a reboot. The connect command should accept the port as an argument, not be hardcoded.
- **Headset not seated properly.** Garbage EEG values produce nonsense words. Players will report "Match-3 says 'anxious' over and over"; the answer is "fix your dry electrode contact."
- **Server thread starvation.** The reader thread runs at default priority. If your server is CPU-bound, dropping it to `BelowNormal` would be safe (only ~57600 baud).
- **GDPR / privacy.** EEG data is biometric. If you're operating in the EU you must disclose collection. The shipped code DOES persist `LatestEEGRaw` on the `PlayerObject` and exposes it to triggers. Don't log it to disk without consent.
- **Audio resource leaks.** The NAudio playback is recreated per stimulus; the `PlaybackStopped` callback disposes. If the stimulus interval drops below MP3 length, the NAudio resource churn may produce clicks. Increase `IntervalMs` (server `EEGSerialManager.IntervalMs`, default 5000) if you hear them.
- **Words can be triggering content.** The CSV is operator-controlled; review the word list for tone before exposing to players. The system has no built-in profanity filter.

## Tuning the experience

| Goal | Tuning lever |
|---|---|
| Reduce EXP firehose | Lower column-B values across the CSV uniformly. |
| Make rare words more impactful | Sparse the CSV near the extreme negative/positive raw values; the wrap fallback makes "common" values produce common words and "rare" values surface rare ones. |
| Match Mir2 lore | Replace generic English words with Mir2 lore terms ("Bichon," "EvilMir," "Zumastatue"). |
| Add visual feedback per intensity | Color the `_label.ForeColour` based on EXP — e.g., low=DimGray, medium=White, high=Gold. ~3 LOC in `ShowStimulus`. |
| Multi-language | Use Unicode in column C. The client font (`Settings.FontName`) must support the glyphs. |

## Quick smoke test

1. Plug in the headset, note the COM port.
2. Start server, log into the game.
3. Run `@EEG 3` (or whatever the port is). Expect `EEG connected on COM3.` in chat.
4. Wait 5 seconds. A word should flash above the HUD; the player's EXP bar should tick up.
5. Disconnect with `@EEG 0` or close the port; the stream should stop within one tick.

If you see no word flashes after 10 seconds despite a successful connect: the reader thread isn't getting valid TGAM packets. Most common cause is paired-but-not-streaming Bluetooth — the headset is on standby. Tap it / re-seat the electrode.
