# n8n Content Engine: Setup & How It Works

Topic/keyword in a Google Sheet -> Gemini writes blog + YouTube metadata + scene script -> QA gate -> **you approve** -> video clips (Omni Flash -> Veo fallback -> manual Flow fallback) -> ffmpeg joins clips and burns captions -> **you approve again** -> YouTube Shorts upload.

## 1. What's in this pack

| File | Purpose | Runs |
|---|---|---|
| `CE-00_Error_Handler.json` | Logs failures to the `Errors` tab and emails you | on failure |
| `CE-01_Content_Generation.json` | Gemini content + QA gate + approval email | every 5 min |
| `CE-02_Video_Clips.json` | One scene clip per run: Omni -> Veo -> manual | every 5 min |
| `CE-03_Manual_Clip_Pickup.json` | Collects clips you drop in Drive | every 10 min |
| `CE-04_Assemble_Video.json` | ffmpeg: normalise, join, captions, upload to Drive | every 10 min |
| `CE-05_Publish_YouTube.json` | Uploads to YouTube | every 15 min |
| `Sheet_*_tab.csv` | Headers (+ sample rows) for the 3 sheet tabs | import once |

Each workflow has a yellow **sticky note** explaining how it works.

## 2. The status flow (the sheet *is* the control panel)

`Draft` -> you set **`Ready`** -> `Generating` -> `Pending Approval` -> you set **`Approved`** -> `Rendering` -> `Clips Ready` -> `Assembling` -> `Video Ready` -> you set **`Publish`** -> `Publishing` -> `Published`

| Status | Meaning | Your action |
|---|---|---|
| `Ready` | Brief is waiting for Gemini | set this to start |
| `Pending Approval` | Passed QA | read/edit, then set `Approved` (or `Rejected` to park it) |
| `Rendering` | Clips being made, one per run | none |
| `Awaiting Manual Clips` | Omni + Veo failed (or manual chosen) | make clips in Flow, upload to Drive |
| `Video Ready` | Final MP4 in Drive | watch it, set `Publish` |
| `QA Failed` | 3 attempts failed QA | edit brief/template, set `Ready`, `attempts` = 0 |
| `Error` / `Publish Failed` | See `error` column | fix, then set the status noted below |

Retry rules: `Error` after Gemini -> set `Ready`. `Error` during assembly -> set `Clips Ready`. `Publish Failed` -> set `Publish`.

**Video provider (column `video_provider`):** `omni_api` (default) | `veo_api` (skip Omni) | `manual_drop` (skip both, email me prompts). Whatever you pick, a failure falls through to the next option.

## 3. One-time setup

### 3.1 Google Sheet
1. Create a new Google Sheet with **three tabs named exactly**: `Content`, `Templates`, `Errors`.
2. In each tab: *File -> Import -> Upload* the matching `Sheet_*_tab.csv` -> "Replace current sheet".
3. In `Content`, add a dropdown (Data -> Data validation) on column B (`status`) with: Draft, Ready, Generating, QA Failed, Pending Approval, Approved, Rejected, Rendering, Awaiting Manual Clips, Clips Ready, Assembling, Video Ready, Publish, Publishing, Published, Publish Failed, Error. **Spelling and capitals must match exactly.**
4. Copy the Sheet ID from its URL (`/spreadsheets/d/<THIS PART>/edit`).

### 3.2 Google Drive
Create two folders: **Manual Clips** and **Final Videos**. Copy each folder ID from its URL.

### 3.3 Credentials in n8n (Credentials -> Create)
| Name to use | Type | Notes |
|---|---|---|
| `Gemini API Key` | **Header Auth** | Name: `x-goog-api-key` · Value: your key from Google AI Studio. Omni and Veo need billing enabled on this key's project. |
| `Google Sheets account` | Google Sheets OAuth2 | |
| `Google Drive account` | Google Drive OAuth2 | |
| `Gmail account` | Gmail OAuth2 | for alerts/approval emails |
| `YouTube account` | YouTube OAuth2 | |

For the Google OAuth ones: create an OAuth client in Google Cloud Console, enable the Sheets, Drive, Gmail and YouTube Data v3 APIs, and add n8n's callback URL shown in the credential screen as an authorised redirect (for a local server this is usually `http://localhost:5678/rest/oauth2-credential/callback`). While the consent screen is in *Testing* mode, add your Google account as a test user.

### 3.4 Server settings (you already proved ffmpeg works)
- The **Execute Command** node must be enabled (`NODES_EXCLUDE="[]"` on newer n8n).
- `node`, `ffmpeg` and `ffprobe` must be on the PATH of the account that runs n8n. (`node` is used for a tiny folder-listing command so the workflows work on Linux, macOS and Windows.)
- `workDir` must be **inside n8n's allowed files folder** (default `~/.n8n-files`). Use forward slashes, even on Windows: `C:/Users/you/.n8n-files/content-engine`.
- On Windows the folder and n8n should be on the same drive (the commands use `cd "<folder>" && ...`).

### 3.5 Import and configure
1. Import the 6 JSON files (*Workflows -> Import from file*). Keep them **inactive** for now.
2. In **every** workflow open the **Config** node and set: `sheetId`, `notifyEmail`, `workDir`, `manualFolderId`, `finalFolderId`. Tip: edit it once, copy the node (Ctrl+C), and paste it over the others, or find/replace the placeholders in the JSON files before importing.
3. Check the model names in Config against Google's current docs (they change): `textModel`, `omniModel`, `veoModel`.
4. Open each node that shows a red/orange credential warning and select your credential.
5. In workflows CE-01 to CE-05: *Settings -> Error workflow* -> **CE-00 | Error Handler**.
6. Activate CE-00 first, then the rest as you test them.

## 4. Test plan (do it in this order, using *Run now* / Execute workflow)

1. **CE-01:** set the sample row in `Content` to `Ready` -> run. Expected: `Generating` -> `Pending Approval`, blog/script filled in, email arrives. Read `script_json`: those are your 4 scene prompts.
2. **Free dry-run of CE-04/05 (no API spend):** make four fake clips with ffmpeg so you can test assembly and upload for free:
   ```
   mkdir <workDir>/post-001
   ffmpeg -f lavfi -i testsrc2=size=720x1280:rate=24:duration=8 -f lavfi -i sine=duration=8 -c:v libx264 -pix_fmt yuv420p -c:a aac -shortest <workDir>/post-001/scene_1.mp4
   ```
   Repeat for `scene_2.mp4`, `scene_3.mp4`, `scene_4.mp4`. Set the row's status to `Clips Ready` -> run **CE-04** -> check `Video Ready`, the Drive file and the captions. Then set `Publish` -> run **CE-05** (privacy is `private` by default).
3. **Manual fallback:** delete the fake clips, set the row to `Approved` and `video_provider` = `manual_drop` -> run **CE-02** -> you get the prompts email -> generate in Flow with your Plus plan, upload as `post-001_s1.mp4` etc. -> run **CE-03** -> `Clips Ready`.
4. **Paid path, one clip first:** set `video_provider` = `omni_api` and run **CE-02** once. It makes exactly **one** 8-second clip. Check the file landed in `workDir/post-001/scene_1.mp4` and `provider_used` says `s1:omni`. Then let the schedule do the rest.
5. Turn all six workflows on.

## 5. Cost & safety knobs
- `maxVideosPerDay` (default **1**) stops a NEW video from starting once today's quota is used. `maxPublishPerDay` does the same for uploads.
- Clips already on disk are **never regenerated**, so re-running is free.
- A row is locked for 20 minutes while a clip is generated, so overlapping runs can't double-bill.
- Cost estimate: roughly **$0.10 per second** of 720p Omni video (verify on Google's pricing page): about **$3 per 32-second Short**. Use `manual_drop` with your Plus credits when you want it free.
- Failed Omni/Veo attempts that already generated a clip but failed to *download* it can be billed once and then retried on the next provider.

## 6. Troubleshooting
| Symptom | Likely cause |
|---|---|
| Execute Command node missing/disabled | enable it (section 3.4) |
| "ffmpeg is not recognized" / "command not found" | PATH of the n8n process; restart n8n after installing |
| "The file is not writable / access not allowed" | `workDir` is outside the allowed files folder |
| Row stuck in `Generating` / `Assembling` / `Publishing` | a run crashed; check the email/Errors tab, set status back one step |
| Sheets node: "column not found" | header spelling in the sheet differs from the CSV templates |
| Omni 400/404 | model name changed; update `omniModel` from Google's Omni docs |
| Veo shows safety block | rewrite that scene's `visual_prompt` in `script_json`, set status back to `Approved` |
| YouTube video private | normal for unaudited API projects; flip it public in Studio or apply for the audit |

## 7. Honest limits of this first version
- **Tested for real:** all JavaScript logic (30 checks: QA gate, daily cap, locking, scene selection, fallback routing helpers, parsers) and the full ffmpeg pipeline (probe -> normalise -> join -> burned captions -> 1080x1920 H.264/AAC) on real video.
- **Not tested inside a live n8n:** node parameter shapes for Sheets/Drive/Gmail/YouTube (n8n changes these between versions) and the live Omni/Veo/Gemini calls, which I built from Google's current docs. Expect to touch up a node or two on first import; the checklist in section 4 finds problems cheaply.
- Caption timing is estimated from clip length (4-word chunks). If you want exact word timing, add a Whisper step later.
- YouTube's "altered/synthetic content" flag can't be set through the n8n node; set it in Studio before going public.
- Not included yet: WordPress publishing, TikTok/Instagram, ElevenLabs voice (Omni/Veo already generate audio), 16:9 renders. Each can be added as another branch after CE-04 or CE-05.
