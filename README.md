# Master AI Prompt Log & Workflow Analysis

**Course:** AME 294: Games and AI — Creating Games with Artificial Intelligence  
**Assignment:** Portfolio Game 2 — Vibe-Coded Browser Game  
**Student Name:** Ying-Ju Chen  
**Project Title:** Babysitter  
**Repository URL:** [https://github.com/yche1364-YJ/game-babysitter](https://github.com/yche1364-YJ/game-babysitter)  
**Play Online (GitHub Pages):** [https://yche1364-yj.github.io/game-babysitter/](https://yche1364-yj.github.io/game-babysitter/)  
**Itch.io URL (Optional Bonus):** —

> **Babysitter** is a cozy typing game built on a Chinese pun: 打蚊子 (swatting mosquitoes) sounds almost the same as 打文字 (typing words). Mosquitoes fly at the crib at night and noises float in during the day. The player types each pest's sound to stop it. Survive seven nights and seven days to earn a Babysitter Certificate. A Zhuyin mode lets players type Zhuyin key positions instead of English.

![Title screen](screenshots/title.png)

---

## 1\. Toolchain & AI Session Inventory

| Category | Primary Tool / Platform | Model Version / Specification | Purpose in Project |
| :---- | :---- | :---- | :---- |
| **Code Scaffolding Agent** | Claude (Cowork mode, Claude desktop app) | `claude-opus-5-5` (identifier shown in the session) | Concept write-up, SVG mockups, single-file `index.html` with an SVG scene and a `requestAnimationFrame` game loop |
| **Logic & Debugging Agent** | Claude (same session) | `claude-opus-5-5` | Typing/lock-on logic, enemy behaviours, night/day loop, scoring, leaderboard, Zhuyin key mapping, bug fixes, automated play-testing with a Playwright typing bot |
| **Audio / SFX Generator** | ElevenLabs | ElevenLabs Sound Effects | Day hit "pop", night "slap", baby crying on game over |
| **Music / Atmosphere** | ElevenLabs | ElevenLabs Sound Effects | 10-second "cute and cozy" looping background track |
| **Visual Asset Pipeline** | Claude, drawing vector art directly as inline SVG code | `claude-opus-5-5` | Baby, crib, mosquitoes, flies, cockroach, mouse, day-noise icons, gauge, certificate. Flat style based on a reference image I supplied. |
| **Fonts** | Google Fonts | Fredoka, Huninn | Rounded UI font; cute rounded font for Zhuyin |
| **Online leaderboard** | Supabase (free plan, Postgres + REST API) | — | Shared leaderboard for the GitHub Pages version; players never sign in |
| **Audio processing** | Claude (ffmpeg + Python in its sandbox) | — | Trimming silence, fades, volume, pitch, loop check, embedding audio in `index.html` |

---

## 2\. Code Development Prompts (Reward — Damage — End Loop)

*My prompts were written in Chinese. They are shown here in English translation, kept as close to the original wording as possible.*

### 2.1 Initial Setup & Boilerplate Scaffolding

* **Date / Session:** `2026-10-03`  
* **Agent Used:** Claude (Cowork)  
* **Target Objective:** Turn the idea into a concept, then build a playable browser game with a title page, a night level and a day level.

#### Exact Prompt Submitted:

Concept:

> I want to make a web game. Mom "types words" (打文字) to get the kid to sleep, so it's a shooter where you type. The closer the mosquito (蚊子) gets, the more the baby cries, so I have to kill it before it bites. Smooth out the logic and give me a short English summary.

First build, after about ten rounds of mockups and art direction:

> OK, build it. It's a browser game with three pages: the home page, then level 1, and when time runs out level 2 (daytime). Levels 1 and 2 alternate. Help me pick timings so players don't get bored.

#### Agent Output Summary & Key Changes:

* One self-contained `index.html`: inline CSS, an SVG scene (960×600 viewBox, scales to the window), HTML overlays for the title, cards, pause and game-over screens, and a single `requestAnimationFrame` loop. No libraries.
* All tunable numbers were put in one `CONFIG` object at the top so the game can be edited later (including with other AI tools).
* It ran on the first try. The agent's own screenshot test found one bug immediately: every overlay was visible at once (see Incident 1).

---

### 2.2 Implementing the Core Gameplay Triad (Reward — Damage — End)

* **Date / Session:** `2026-10-03` to `2026-10-06`  
* **Agent Used:** Claude (Cowork)  
* **Target Objective:** Program the three foundational gameplay pillars.

#### A. Reward Mechanic Prompt:

> Should there be a next level? It becomes daytime, with construction noise upstairs and noise outside the window, still as words. The baby is awake and crying; you type away the noises (same gameplay) so the baby stops crying and laughs.

> And at the end, a ranking of everyone who has played: "you are babysitter #N".

> Each round, each level should add a new enemy.

* **Implementation Outcome:** Every word typed is a hit: a burst ("SLAP!" at night, "POW!" by day), +10 × round points, and by day +3% happiness. Surviving a level adds +100 × round. Each round introduces a new character with a picture card. Game-over shows the score and the player's rank ("You're the #2 babysitter out of 12"), saved to a shared leaderboard.

#### B. Damage & Hazard Mechanic Prompt:

> Make the penalty 10%, and describe the meter as levels from happy to crying and asleep to awake, not seconds.

* **Implementation Outcome:** A ring gauge shows the baby's state in words (Deep sleep → Asleep → Restless → Stirring → Waking!, and Giggling → … → Crying!) plus a percentage. A pest reaching the baby costs 10%, a wrong key 2%, and pests inside the red danger ring drain it every second. The baby's face changes with the meter.

#### C. End State & Win/Loss Condition Prompt:

> A clear ending: survive seven days to win, then a short animation of the baby happily climbing out of the crib. Then a babysitter certificate.

* **Implementation Outcome:** Losing (meter at 0%) shows "The baby woke up" or "The baby started crying" with stats, a name box, Save score, Play again, Leaderboard and Home. Winning day 7 plays the climb-out animation with confetti, then a Babysitter Certificate with the player's name, date, score and a CERTIFIED seal.

---

### 2.3 Wiring Up the Web Audio API & Audio Triggers

* **Date / Session:** `2026-10-06`  
* **Agent Used:** Claude (Cowork)  
* **Target Objective:** Load my audio files, attach them to game events, and unlock audio after a user gesture.

#### Exact Prompt Submitted:

> Add sound effects and a looping background track — here are the files. Play this crying sound when the player loses. Use this sound for POW by day and adjust where it starts. Use this one for slap, quieter. Make sure every daytime hit is the pow and every night hit is the slap.

#### Implementation Outcome:

* One `AudioContext`, created and resumed on the player's first click or key press (browser autoplay policy). The cry and hit sounds are pre-decoded at that moment so they play with no delay.
* The music loops gaplessly through an `AudioBufferSourceNode` with `loop = true` (an `<audio loop>` tag leaves a gap). A low-pass filter and gain change with the scene: muffled and quiet at night, bright by day, ducked when paused or after losing.
* Hits play the recorded pop (day) or slap (night). Losing plays the baby crying. Separate volume settings: `musicVolume`, `popVolume`, `slapVolume`, `cryVolume`.
* Smaller cues (key click, lock-on ping, baby whimper, countdown bell, giggle, damage tone, win jingle) are synthesized with Web Audio oscillators.
* The audio is embedded in `index.html` as base64 so the single file works offline. Copies are in `assets/audio/`.

---

## 3\. Generative Audio & Sound Design Prompts

### 3.1 Sound 1: Reward SFX

* **Target Event:** A pest is stopped — daytime "POW!" (and a second sound for the night "SLAP!")  
* **Audio Generator:** ElevenLabs Sound Effects  
* **Exact Prompt / Acoustic Descriptors:**  

  Day pop:
  > One sharp and crisp bubble pop: "pow!" Clean, punchy, playful, and cartoon-like. Single sound only, very short, no reverb or background noise.

  Night slap:
  > One sharp: "slap" cute, playful, and cartoon-like. Single sound only, very short, no reverb or background noise.  

* **Iterations & Refinements:**  
  * Pop: the 1-second file had 0.27 s of silence before the sound. It was trimmed to 0.3 s so the pop lands with the visual burst, given a short fade, and lowered from 0.9 to 0.45 after play-testing ("the pop can be quieter").
  * Slap: this took six rounds. The raw slap felt too sharp against the soft music (about a third of its energy was above 3 kHz). Filtered and synthesized versions were rejected ("still a slap, just not so loud, more cartoonish and playful"). Final: the recording sped up 1.2× (higher, shorter, more cartoon-like), highs rounded off, 0.13 s, volume 0.35.  
* **Exported Filename:** `/assets/audio/reward_day_pop.wav`, `/assets/audio/reward_night_slap.wav`

---

### 3.2 Sound 2: Damage SFX

* **Target Event:** A pest reaches the baby (−10%), and a pest enters the red danger ring  
* **Audio Generator:** Created by Claude in code (Web Audio API oscillators), under my direction  
* **Exact Prompt / Acoustic Descriptors:**  

  There was no separate audio-generator prompt. The damage sound was built by the coding agent as part of the game, following my build request in Section 2.1 and my sound request in Section 2.3 ("I want to add some sound effects"). I reviewed it in play-testing and kept it quiet so it doesn't compete with the music and the crying sound.  

* **Iterations & Refinements:** The hit is a short falling sawtooth "ouch" tone with a shake of the baby. A soft two-note whimper plays when a pest enters the danger ring, limited to once every 2.5 seconds so it never stacks.  
* **Exported Filename:** — (generated at runtime)

---

### 3.3 Sound 3: End State SFX

* **Target Event:** Game over — the baby wakes up or starts crying  
* **Audio Generator:** ElevenLabs Sound Effects  
* **Exact Prompt / Acoustic Descriptors:**  

  > making the crying baby sound, soft and not too annoying and more cute not too sharp like baby just starting to cry  

* **Iterations & Refinements:** 5-second clip, converted to mono 24 kHz to keep the page small, with a 0.4 s fade-out because the original ended abruptly. Plays at 0.8 while the music drops to a quiet "game over" level so the cry stands out. Winning uses a synthesized four-note jingle with confetti instead.  
* **Exported Filename:** `/assets/audio/end_baby_cry.wav`

---

### 3.4 (Optional) Ambient Music / Background Soundscape

* **Target Purpose:** Looping background music for the whole game  
* **Tool Used:** ElevenLabs Sound Effects  
* **Prompt & Settings:**  

  > Cute and cozy game background music with low-pitched marimba and soft wooden percussion. Warm, bouncy "boom boom" notes, playful and gentle, medium tempo. Avoid high-pitched xylophone, bells, chimes, and crystal sounds. good for night  

  Analysis: 10.0 s, loops cleanly (near-zero samples at both ends), roughly G major with a one-second rhythmic pattern. Converted to 24 kHz to save space.  

* **Exported Filename:** `/assets/audio/bg_music.wav`

---

## 4\. Debugging, Error Recovery & Friction Log

### Incident 1: Every screen showed at the same time

* **Symptom / Error Message:** The agent's first automated screenshot showed the title, the level card, "Paused" and "The baby woke up" stacked on top of each other.
* **Root Cause:** The overlays were hidden with the HTML `hidden` attribute, but the CSS rule `.overlay { display: flex }` overrode it.
* **AI Follow-up Prompt Used to Fix:** None. The agent caught this in its own screenshot check before I saw it.
* **Resolution:** Added `[hidden] { display: none !important; }`.

---

### Incident 2: Only one leaderboard entry after two games

* **Symptom / Error Message:** I played twice but only one score appeared, under the name "0".

  > I played twice; there should be two entries on the leaderboard, but there aren't.

* **Root Cause:** The leaderboard stored one record per player and only replaced it on a new best, so a lower second score was silently dropped. It was a design assumption by the AI, not a crash.
* **AI Follow-up Prompt Used to Fix:** The message above.
* **Resolution:** Each player's record now holds a list of their games (up to 25). The board flattens every game from every player and sorts by score. Old single-score records were migrated.

---

### Incident 3: The name box disappeared after the first save

* **Symptom / Error Message:**

  > After replaying I can't enter a name anymore. Every time the player loses, they should be able to type their own name.

* **Root Cause:** After the first save, the code switched to auto-saving and hid the form, so a second player on the same computer couldn't enter their name.
* **AI Follow-up Prompt Used to Fix:** The message above.
* **Resolution:** The name box now appears after every game, pre-filled with the last name and selected so a new player can type over it. Saving is always the player's choice.

---

### Incident 4: The game could not be beaten, and the AI could not hear its own sounds

* **Symptom:** When I asked the agent to play to the end, it could not "play" like a person, so it wrote a Playwright bot that sends real key presses at a fixed speed. The bot showed that even at 9 perfect keys per second the game was lost on night 7. Separately, the agent cannot hear audio, so its slap-sound edits were judged only from waveforms and missed what I wanted several times.

  > Can you play it once for me, all the way to the end?

  > The goal is that a very fast typist can almost beat it.

* **Root Cause:** The late-round spawn rate ramped to a pest every 0.24 s with up to 16 on screen.
* **Resolution:** Added `minSpawnEvery: 0.46` and `maxOnScreenCap: 11`, then re-ran the bot after each content change. Final: 9 keys/s wins about 3 runs in 4, 7 keys/s reaches the last day. For audio, the agent built small listening pages (with the night music underneath and three hits in a row) so I could choose by ear.

---

### Incident 5: The leaderboard did not save on GitHub Pages

* **Symptom:** Played online from GitHub Pages, the game could not save scores to a shared leaderboard.

  > When I play it online, the leaderboard can't save. I don't want a login, but I still want a leaderboard. Can you do that?

* **Root Cause:** The shared leaderboard used Claude's built-in artifact storage, which only exists when the game is opened on claude.ai. A plain web page on GitHub Pages has no server to store scores.
* **Resolution:** Added a second leaderboard mode that talks to a free Supabase database through its REST API with plain `fetch`. Only I (the developer) have a Supabase account; players never sign in. Each browser gets a random player id, and the database accepts anonymous reads and inserts only, with limits on name length and score range. The agent tested it against a mock server in two separate browsers. Setup steps are in `SUPABASE_SETUP.md`.

---

## 5\. Human-in-the-Loop Curation & Analytical Reflection

> 200–300 words analyzing the collaborative dynamic between you and the AI tools

> **Reflection Prompting Questions to Consider:**
>
> - Where did the AI coding agent accelerate your workflow the most?
> - Where did the AI fall short, hallucinate obsolete API methods, or introduce subtle bugs?
> - How did your personal game design intuition guide decisions regarding pacing, difficulty, and audio volume balancing?
> - How did designing sound effects upfront influence how the game feel evolved?

My original idea was a game about swatting mosquitoes. In Chinese, "typing words" (打文字) and "swatting mosquitoes" (打蚊子) sound very similar, and a typo I made while prompting the AI turned into a new idea: turning typing into the attack is a lot of fun. Besides English, I also added a Zhuyin version, using the phonetic symbols unique to Taiwan.

AI tools helped my development a great deal. I often have ideas but need to see them made right away, and AI can do that. Because results appear immediately, I can adjust them on the spot and refine my original idea. The AI could also revise every part I pointed out. For sound, we first discussed options separately and only applied the result to the game once we agreed. It generated each screen layout for me, and I changed the details from there. Even the ending animation I wanted at the end could be generated right away. For the game mechanics, I asked it for a table of the current levels with their gameplay and an analysis, then adjusted the content directly in that table, which made everything well organized.

Through repeated testing, asking the AI to play-test the game and send me screenshots, and giving it examples, I shaped the interface I wanted. I came up with the ideas and discussed them with the AI; it gave me results to judge and choose from. That let me focus on creative thinking, and it became a very capable assistant. The process sparked a lot of creativity and refined the game into the current version.

---

## 6\. Asset Attribution & Licensing Table

| Asset Filename | Asset Type | AI Model / Source Tool | License / Terms | Prompt / Origin Details |
| :---- | :---- | :---- | :---- | :---- |
| `index.html` | Game code (HTML/CSS/JS) | Claude (`claude-opus-5-5`) | Educational project | Written by the agent from the prompts in Section 2 |
| `reward_day_pop.wav` | SFX (Audio) | ElevenLabs Sound Effects | ElevenLabs Terms of Service (paid Creator plan) | Prompt in Sec 3.1; trimmed and lowered by the agent |
| `reward_night_slap.wav` | SFX (Audio) | ElevenLabs Sound Effects | ElevenLabs Terms of Service (paid Creator plan) | Prompt in Sec 3.1; sped up 1.2× and softened by the agent |
| `end_baby_cry.wav` | SFX (Audio) | ElevenLabs Sound Effects | ElevenLabs Terms of Service (paid Creator plan) | Prompt in Sec 3.3; fade-out added |
| `bg_music.wav` | Music (Audio) | ElevenLabs Sound Effects | ElevenLabs Terms of Service (paid Creator plan) | Prompt in Sec 3.4 |
| Damage / UI cues | SFX (synthesized) | Web Audio API, code by Claude | Part of `index.html` | Made by the agent under my direction (Sec 3.2) |
| Characters, crib, icons, certificate | Vector art (inline SVG) | Claude, drawn as code | Part of `index.html` | Flat style from a reference image I supplied; characters are original |
| Fredoka, Huninn | Fonts | Google Fonts | SIL Open Font License 1.1 | Loaded from fonts.googleapis.com |
| `screenshots/*` | Images | Captured by the agent with Playwright | Same as the project | Screens from the finished game |

---

### Running the game

Play at [yche1364-yj.github.io/game-babysitter](https://yche1364-yj.github.io/game-babysitter/), or open `index.html` in a browser. To publish: Settings → Pages → Branch `main`, folder `/ (root)`. A keyboard is needed. The leaderboard is shared online with no sign-in for players: the Claude-hosted version uses Claude's built-in storage, and the GitHub Pages version uses a free Supabase database (setup steps in [SUPABASE_SETUP.md](SUPABASE_SETUP.md)). Everything adjustable is in the `CONFIG` block at the top of the script.

| Night | Day | Certificate |
|---|---|---|
| ![Night](screenshots/night.png) | ![Day](screenshots/day.png) | ![Certificate](screenshots/certificate.png) |
