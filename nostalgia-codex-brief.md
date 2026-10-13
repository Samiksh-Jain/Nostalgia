# Codex brief: Nostalgia ("Your trip, unpacked.")

This replaces the earlier trip-to-posts brief. Build a NEW app in a fresh Convex project. Product name comes from the APP_NAME env var (default `nostalgia`); never hard-code it.

Brand: the app is **Nostalgia** (wordmark "nostalgia." in lowercase with the red dot). "unpacked." stays as the signature line on the recap itself ("<Name>, unpacked.") and "Unpack" stays as the action verb on buttons. The prototype already uses this split; copy it.

## 0. Read this first: the reference prototype

`reference/nostalgia-prototype.html` is a working, single-file prototype of the core experience. Open it in a browser, tap "Watch the example", then try "Unpack my photos" with real photos.

- It is the **design and behaviour reference**. Match its story structure, card designs, motion, type, the trip-derived colour system and the baggage-tag share card. Where this brief and the prototype disagree, this brief wins.
- Its on-device analysis code (EXIF read, near-duplicate hash, blur score, quality score, bursts, hour histogram, colour palette, route, personas) works and was tested on sample photos. Port it into TypeScript modules rather than rewriting it.
- The example trip in it uses drawn illustrations. Keep it as the landing-page demo, still labelled "Example".

## 1. What it is

Spotify Wrapped, for a trip. Someone picks every photo from a trip (20 to 400). About a minute later they watch a full-screen, tap-through recap story made from their own photos: how many they took, the hour they couldn't stop shooting, their biggest burst, their trip in colour, day by day, their top 5, the blurry one they almost deleted, what got culled, their trip personality, and a baggage-tag summary card. Then they share it: the story as a link to the family group, the tag as an Instagram story, and a ready 4:5 carousel for the feed.

The same works for any batch of photos: a wedding, Diwali, "October". The title pattern carries it: "<Name>, unpacked."

**North star moment:** the user watches their own recap once and immediately sends it to a group chat. Every decision serves that.

**Sprint context:** Growth X Build Sprint, track Virality. Submission is a live URL, a public repo and real numbers, due Saturday 17 Oct before 11 AM IST (aim for 9 AM). The owner tests on his own recent vacation first.

## 2. Why this can win (keep these true)

1. **It's about the person, not the product.** Wrapped spreads because it shows the user to themselves. Every card is a fact about *their* trip.
2. **Every number is real.** Counts, hours, bursts, colours and percentages come from the photos. Nothing invented, no filler stats. If a stat can't be computed, the card is skipped.
3. **Each trip looks different.** The story's colours are mixed from the trip's own pixels (prototype `themeFrom`). A Goa trip is sunset orange and ocean; a Manali trip is pine and snow.
4. **Desi, playful voice.** Persona lines and colour names are Hinglish and Indian ("Sunset dikha, phone nikla.", Haldi, Chai, Jamun, Gulabi). This is what makes a family group reply.
5. **Private by default.** Full photos never leave the phone. Only small previews of the keepers are sent for AI, and deleted after 7 days. Say this plainly in the UI.
6. **The share is the ad.** The recap link plays for anyone without an install or sign-in and ends with "Unpack your trip".

Similar apps exist (Recollect, TripReel, TripChronos) and make map or globe videos from photo locations. Nostalgia is different because it reads the photos themselves (bursts, quality, colour, content), speaks in a personal voice, and outputs a share link plus Instagram-ready files.

## 3. Platform facts to respect (checked October 2026)

- **Target browser is Chrome**, on iPhone first, then Android. On iPhone (outside the EU) every browser, Chrome included, runs on Apple's WebKit engine and uses the iPhone's own photo picker, so Chrome on iPhone handles photo uploads the same way Safari does: it **keeps the capture date** (EXIF DateTimeOriginal, since iOS 16.4) but **strips GPS by default**. From iOS 17 the photo picker has an "Options" menu where the user can include location; confirm this menu appears when the picker is opened from Chrome. So: time-based cards are the core; the route card is a bonus that appears only when ≥3 photos carry GPS. Keep the "tap Options → Location" tip on the picker screen.
- iPhone converts HEIC to JPEG for `<input type="file" accept="image/*">` in Chrome and Safari alike. Desktop Chrome can't decode HEIC; show a clear message ("open this on your phone, or export as JPEG").
- Google Photos: apps can only read photos the user picks through the Picker API (full-library access ended 31 Mar 2025). Not needed tonight; the phone's own picker reaches Google Photos and iCloud Photos already synced to the device.
- Instagram: carousel up to 20 slides; the first slide sets the aspect ratio for all; use 1080×1350 (4:5) for every slide; one caption per carousel; the profile grid shows slide 1 cropped to 3:4, so keep cover text centred. Videos inside a carousel don't count as Reels.

## 4. Architecture (decisions already made)

- **Stack:** TypeScript, React + Vite, Convex (database, file storage, actions, scheduled jobs, hosting). Mobile-first web app. Primary target: Chrome on iPhone. Second: Chrome on Android. Also check Safari on iPhone before each deploy, since some people will open shared links there.
- **On-device pass (no network):** for each photo, read EXIF date + GPS (exifr), decode once at reduced size, compute: mean luminance, contrast, colourfulness, Laplacian variance (blur), 64-bit dHash, colour samples, and make a 360px thumbnail. Videos: count and duration only (not uploaded). Run 2 workers in parallel. Then build the report: near-duplicate groups, keepers, top 5 (spread out in time), outtake, bursts, hour histogram, days, palette (k-means in Lab, k=5, percentages that add up to 100), route, persona. All of this exists in the prototype (`analyzeDrawable`, `buildReport`, `kmeansPalette`, `themeFrom`).
- **AI pass (Convex action):** upload only the keepers' 512px previews (cap 40, EXIF stripped, JPEG), in batches of 10 per vision call, in parallel. Model from env `VISION_MODEL`, key `OPENAI_API_KEY`. Strict JSON per photo: `tags` from a fixed list (food, chai_coffee, dessert, sunset_sunrise, beach, mountain, water, forest_nature, street_city, landmark, place_of_worship, group_of_people, selfie, portrait, night, vehicle, animal, indoor), `hero_score` 0-10, `candid_score` 0-10, `one_line` (a factual description of what is visible). Never identify, name or describe people beyond counts and visible actions. Then one text call over the `one_line`s to write: 3 trip-title suggestions (only used if the user left the name blank; no place names unless the user typed one), and 3 Instagram captions (see §7).
- **Merge:** AI `hero_score` becomes 40% of the top-5 ranking (on-device quality 60%). Tags feed the "Couldn't stop photographing" card and the content personas. If AI fails or times out (25 s), the story still plays with on-device cards only. Never block the story on AI.
- **Streaming:** show the story as soon as the on-device report is ready (seconds). AI cards slot in when they arrive; if the user is already past that point, add them before the summary.
- **Mock mode must be loud:** without `OPENAI_API_KEY` the AI cards and captions are replaced by a visible "Demo mode: AI captions off" banner on every screen and a warning in admin. Never silent.

## 5. The story (card by card)

Full-screen 9:16, tap right = next, tap left = back, hold = pause, progress bars on top, auto-advance. Copy and layout exactly as in the prototype unless noted. "Skip if" means the card is left out, not shown empty.

| # | Card | Data | Skip if |
|---|------|------|---------|
| 1 | **Intro**: "<NAME>, *unpacked.*" over a drifting mosaic of the best shots; chips for date range, days, photo count | name typed (or AI title), EXIF dates | never |
| 2 | **Count**: "You took 312 photos. In 4 days. That's one every 9 minutes you were awake." + videos and footage length | counts, active days × 16 waking hours | never (no dates → say the timeline is skipped) |
| 3 | **By the hour**: radial 24-hour clock, peak hour in the centre, line by peak bucket ("Golden hour hit different."), "% of photos between 6 PM and 7 PM" | EXIF hours | <6 dated photos or <50% dated |
| 4 | **Burst**: "38 photos in 11 minutes", fanned photos, "DAY 2 · SAT 13 OCT · 6:42 PM" | densest 15-min window | burst < 5 photos |
| 5 | **NEW (AI) Couldn't stop photographing**: top 3 tags as big counts ("Sunsets ×23 · Chai ×14 · Food ×31") with one photo each | AI tags on keepers | AI unavailable or top tag < 3 photos |
| 6 | **Colour**: palette bar + 5 named colours with % ("Ocean 34%, Haldi 21%…"), "This whole recap is painted with them." | k-means palette | never |
| 7 | **Route**: drawn route, km, spots | GPS | <3 GPS photos or <1 km |
| 8 | **Day by day**: best photo per day, bar per day, "Day 2 was the big one." or "Every day pulled its weight." when days are close | EXIF dates | <2 active days |
| 9 | **Top 5**: #5 to #2 revealed, then #1 large with "The one." and its frame number | quality + AI hero score, spread ≥40 min apart | <3 keepers |
| 10 | **Outtake**: the most colourful blurry shot as a polaroid, "Probably the best memory." (in the app, prefer AI `candid_score` when present) | blur + colour (+ AI candid) | none blurry or candid |
| 11 | **Cull**: "312 in. 24 keepers." with Keepers / Near-copies / Blurry / Too dark meters | dHash groups, blur, darkness | never |
| 12 | **Personality**: archetype name, Hinglish line, English gloss, 3 real stat chips | persona rules (§6) | never |
| 13 | **Summary tag**: baggage-tag card with 3-letter code, hero photo, name, photos/days/peak/keepers, persona, palette strip, "make yours" | everything above | never |

After the summary: "Share my recap", "Get Instagram kit", "Replay", "New trip".

## 6. Personas (deterministic, explainable, fun)

Keep the prototype's scoring (each persona score is a measured share divided by a baseline; highest wins; "Storyteller" is the default at 1.0). Store which rule won and its numbers so the chips are honest. Existing: Golden Hour Chaser, Early Riser, Night Owl, Burst Mode Royalty, Archivist, Curator, Colour Hunter, Explorer, Storyteller.

Add these AI-tag personas (score = tag share ÷ baseline):

- **The Foodie Archivist** ("Pehle photo, phir khana."): food + chai_coffee + dessert share ÷ 0.18
- **The Squad Captain** ("Group photo ke bina trip adhoori."): group_of_people share ÷ 0.25
- **The Landscape Romantic** ("Pahad dekhe, dil haar gaye."): mountain + forest_nature + water + beach share ÷ 0.45
- **The City Wanderer** ("Har gali ek kahaani."): street_city + landmark share ÷ 0.3

Keep the full list in `personas.json` (id, name, hinglish line, english gloss, rule, baseline) so copy can change without code.

## 7. Instagram kit and captions

- **Carousel (4:5, 1080×1350 JPEG):** slide 1 cover (aura gradient + trip name + "unpacked." over the #1 photo), then up to 8 keepers (top 5 first, then by score, no near-duplicates, spread over days), then a stats slide, then an end card "nostalgia. · make yours" (removed with Nostalgia+). Photos fill the frame with a focal crop from the analysis; never stretch, never crop through a face.
- **Story set (9:16, 1080×1920 PNG):** render the Summary tag, the Personality card and the Top 5 card as images.
- **Rendering:** draw these export images with Canvas 2D from the report data (deterministic, no DOM screenshots). Load the brand fonts with `document.fonts.load` before drawing.
- **Captions:** 3 options per carousel: "Seedha", "Dramatic", "Masti". Language toggle English / Hinglish / Hindi, default Hinglish. Each must mention at least one specific visible detail from the `one_line`s. Place names only if the user typed them. 5-8 hashtags from the trip's tags + typed place, no spam bundles. Bad: "Beautiful moments from my trip! #travel #instagood". Good (only if chai and an unplanned stop are visible or typed): "Teen din, ek hi plan: jahan chai mile wahin ruk jao."
- **Save / share:** `navigator.share({ files })` when `navigator.canShare({ files })` is true (opens the phone share sheet, Instagram appears there); otherwise download a zip. Copy the chosen caption to the clipboard on tap and say so.

## 8. The share link (the virality loop)

- "Share my recap" asks once: "This makes a link anyone with it can open. It shows N of your photos." On yes, upload only the photos the story uses (≤20 previews at 1080px, EXIF stripped) plus the report JSON, and create `/u/<random-id>` (unguessable, 16+ chars).
- `/u/<id>` plays the same story for anyone, no sign-in, no install. Last card: "Unpack your own trip" button to the landing page.
- OG tags: title "<Name>, unpacked.", image = the rendered Summary tag (1200×630 crop variant), so the WhatsApp preview looks like the tag.
- Owner can delete the link anytime from the trip screen. Links expire after 30 days.
- One-tap "Send to WhatsApp" opens `https://wa.me/?text=` with the title and link.

## 9. Nostalgia+ (honest fake door)

Button on the summary and kit: "Nostalgia+ ₹99 per trip: no end card, HD exports, Recap Reel video, Squad mode." Tapping asks for a WhatsApp number and says: "Payments aren't live yet. We'll message you when they are." Log `unlock_tap` and `unlock_submitted`. No payment code tonight.

## 10. Analytics (real events only; these are the sprint numbers)

`landing_view`, `example_played`, `example_completed`, `picker_opened`, `photos_selected` (count, videos), `ondevice_report_ready` (seconds, photos, keepers, dated %, gps %), `ai_ready` (seconds, ok/failed), `story_card_view` (card id), `story_completed`, `replay`, `share_link_created`, `share_link_view` (unique per device), `share_link_completed`, `share_link_make_yours_click`, `kit_opened`, `kit_shared`, `caption_copied`, `unlock_tap`, `unlock_submitted`, `delete_trip`. No photos, names or EXIF in analytics. Password-protected `/admin` (env `ADMIN_PASSWORD`): totals by day, funnel (landing → example → picked → story completed → shared → link views → make-yours clicks), median time to story, AI cost per trip, failures list.

**K-factor** = make-yours clicks that reach `story_completed` ÷ share links created. Show it in admin; it is the Virality track's headline number.

## 11. Privacy and data

- Full-resolution photos never leave the device. Previews sent for AI and share links are EXIF-stripped.
- Delete AI previews 7 days after the last activity (Convex scheduled job). Delete share links after 30 days or on request.
- No accounts. A device id in localStorage plus "Send my trip link to my WhatsApp" so they can reopen it.
- One plain sentence on the picker screen: "Your photos stay on your phone. Only small previews of your best shots are used to write captions, and they're deleted in 7 days."

## 12. Build order (deploy and commit after each step; update STATUS.md each time)

Tonight (Wed night):
1. `ARCHITECTURE.md` (short). Scaffold React + Vite + Convex. Copy `reference/nostalgia-prototype.html` into the repo.
2. Port the on-device analysis and report (`analysis.ts`, `report.ts`, `palette.ts`, `personas.ts`) with unit tests on a fixed set of 30 test JPEGs (tests check: dates read, near-duplicates grouped, blurry detected, palette % sums to 100, burst found).
3. Port the story player and all on-device cards, the landing page and the example trip. Deploy. **This alone is a shippable product.**
4. Share link `/u/<id>` with OG image and WhatsApp share.

Thursday:
5. AI pass (tags, hero/candid scores, captions, titles), the "Couldn't stop photographing" card, AI personas, loud mock mode.
6. Instagram kit export (carousel + story set), Web Share with files, zip fallback.
7. Analytics + `/admin` + Nostalgia+ fake door + delete flows + cleanup jobs.

Friday (only if the above is solid):
8. Recap Reel: record the story as a 9:16 video (canvas + MediaRecorder; pick MP4 when `MediaRecorder.isTypeSupported` allows it, as Instagram expects MP4). Until it ships, leave the Reel line out of the kit. The app never tells users to screenshot or screen-record anything; Share does the work.
9. Squad mode (like Spotify's Wrapped Party): friends add their photos to the same trip via a link; a joint recap adds "Who shot the most" and a combined palette.

## 13. Acceptance checks (the owner runs these as a user)

1. On my iPhone in Chrome, I pick 100+ of my real trip photos and the first story card plays within 60 seconds of tapping "Unpack". Nothing full-size is uploaded (check the network tab).
2. The photo count, date range, days, peak hour and burst time match my camera roll.
3. Near-copies and blurry shots in the cull card are really near-copies and blurry. The top 5 has no two near-identical shots.
4. The palette percentages add up to 100 and the story's colours visibly come from my trip.
5. The persona's chips show real numbers that explain why I got it.
6. With location turned on in the picker, the route card appears; with it off, it doesn't.
7. "Share my recap" gives a link that plays on another phone without sign-in, shows the tag as the WhatsApp preview, and ends with "Unpack your own trip".
8. The kit exports 4:5 slides at exactly 1080×1350 and 9:16 stories at 1080×1920; the share sheet offers Instagram on my iPhone in Chrome and on an Android phone in Chrome.
9. Captions mention something that is actually in my photos and no place I didn't type.
10. Every event shows in `/admin` within a minute; no fake numbers anywhere.
11. With the AI key removed, the story still plays and the demo-mode banner is visible.
12. Hindi and Hinglish text renders correctly in the app, the exports and the share page.

## 14. What the owner provides (list in README and STATUS.md)

- `OPENAI_API_KEY`, `VISION_MODEL`, `ADMIN_PASSWORD`, `APP_NAME`.
- 100+ real photos from his own recent trip for testing (with permission from people in them before any of it becomes a public example).
- A WhatsApp number for the Nostalgia+ confirmation message, if wanted.
