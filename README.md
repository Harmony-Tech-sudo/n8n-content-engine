# n8n AI Content Engine

Practice project: an automated content pipeline on self-hosted n8n.
A topic and keyword in a Google Sheet become a blog post, YouTube metadata and a
captioned vertical video, published to YouTube Shorts with two manual approval gates.

**Stack:** n8n · Gemini (text) · Gemini Omni Flash → Veo 3.1 → manual Flow fallback (video) · ffmpeg · Google Sheets/Drive/Gmail · YouTube API

| Workflow | Purpose |
|---|---|
| `CE-00_Error_Handler.json` | Logs failures to the sheet and sends email alerts |
| `CE-01_Content_Generation.json` | Gemini content generation, QA gate (length, keyword, banned phrases), approval email |
| `CE-02_Video_Clips.json` | Generates scene clips with Omni → Veo → manual fallback, with daily cost cap |
| `CE-03_Manual_Clip_Pickup.json` | Collects clips made by hand in Google Flow from a Drive folder |
| `CE-04_Assemble_Video.json` | ffmpeg: normalise, join, burn captions, upload to Drive |
| `CE-05_Publish_YouTube.json` | Uploads approved videos to YouTube Shorts |

The `Sheet_*_tab.csv` files are the Google Sheet tab templates, and `README_Setup.md` covers
setup, the status flow and a test plan.

**Status:** first version. Logic and the ffmpeg pipeline are tested; the n8n node
settings and live API calls still need testing on first import.
