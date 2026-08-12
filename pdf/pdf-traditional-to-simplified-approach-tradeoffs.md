# Converting Traditional Chinese to Simplified in PDFs: Why We Ended Up With a "Combined Approach"

Batch-converting the traditional Chinese characters in a Hong Kong stock interim report to simplified sounds as simple as find-and-replace. Once we actually tried it, we found PDF is an extremely unfriendly format for "changing characters": the glyph, position, spacing, images, and font encoding of text are scattered across several structural layers, so touching one place easily drags a whole chain of things with it.

This article lays out the pitfalls we hit and the final decision: the strengths and weaknesses of three technical approaches, why none of them is enough on its own, and why we eventually converged on a combined approach of "handle by case + unified verification".

## 1. First, Understand How a Single Character Gets Drawn in a PDF

To follow the trade-offs later, we first need a basic mental model: a Chinese character we see in a PDF is actually the result of three things working together.

- Content stream: a series of drawing instructions, something like "at coordinate (x, y), draw glyph number 0x27F3 using the current font". It governs order, position, spacing, and layout direction.
- Embedded font: a "glyph table" that tells the renderer what glyph number 0x27F3 looks like (the outline curves).
- ToUnicode mapping: a "translation table" that tells copy-paste/search which Unicode character glyph number 0x27F3 corresponds to (for example U+5831 "報").

To use an analogy, the content stream is like a "menu ordered by number" — it only says "serve dish No. 27", and to find out what dish No. 27 actually is, you have to flip through the font, which is the recipe book. Once we understand what each of these three layers is responsible for, we can see clearly: to change one character, which layer should we actually touch.

## 2. Approach A: Paste the Replacement Character as New Text

This is the approach we built first, and the one currently working. The idea is straightforward: first use redaction to delete the original traditional glyph from the page, then use a high-level API (TextWriter) to write the simplified character as a brand-new text object appended to the end of the content stream.

Its biggest advantage is "simple to implement, strong controllability":

- The whole flow uses PyMuPDF's high-level API, without touching the low-level PDF syntax.
- We embed the new font ourselves and control its encoding ourselves, so we're not affected by the original font missing cmap / ToUnicode.
- Verification capability is the strongest. Because the original character is truly deleted and the new one is added 1:1, we can build a triple gate of "rendered pixels + character multiset + image fingerprint". In practice the total character count is conserved page by page (269746 in, 269746 out), and this strong verification is exactly what A's structure naturally provides.

But that very structure of "start fresh and paste new characters" brings a string of unavoidable problems:

- **The copied text comes out in scrambled order — this is A's most fatal flaw**. Because the new characters are "pasted" at the end of the content stream, on the page the characters are all in their original positions, neatly arranged, and read perfectly fine to the human eye; but the moment you select and copy, the text-layer order you get is completely scrambled. In other words, visual correctness and text-layer correctness are split into two separate things under A — it looks right, but what you copy is a pile of misordered characters. Turning on `sort=True` only recovers to the level of "characters are roughly in order, but words get cut apart"; it cannot recover the true reading order. For a PDF that's going to be used for search, excerpting, or fed to downstream programs for parsing, this flaw is almost fatal.
- **The font style "drifts"**. The replacement characters use Noto, which is a completely different font from the original HYQiHei. When they sit side by side, the weight, structure, and stroke style are clearly inconsistent — in an originally uniform passage, the few replaced characters look out of place, as if clipped from somewhere else. Reading down the whole page, the font style wavers, and the visual quality noticeably suffers.
- Size bloat. The content stream grew 63% (889KB → 1447KB), and embedding 3 additional Noto font subsets added another 436KB.
- Redaction causes collateral damage, and all of these are pitfalls we hit in practice: the character box of full-width punctuation presses into neighboring characters (later worked around with `_safe_redact_rect`), vertical-text geometric offsets delete the wrong cell (later changed to just skip), and there's also the "mysteriously missing 1 image" problem in 4 interim reports that remains unsolved to this day.
- The original font can't be deleted. Characters that weren't rewritten still use the original font, so both the old and new fonts have to be kept.

To sum up A in one sentence: two flaws stand out most — the copied text is scrambled (looks right, copies wrong) and the font style drifts, plus size bloat and redaction collateral damage; but it wins on being simple to implement and having the hardest verification.

## 3. Approach B: Directly Modify the Drawing Instructions in the Content Stream

Since pasting new characters scrambles the order, could we instead modify that instruction in the content stream, "draw glyph number 0x27F3", in place to "draw that simplified glyph"? That's approach B.

Its advantages are almost the mirror image of A's:

- Order, position, and spacing are naturally correct, because we didn't add any new text objects.
- No redaction, so images and vector primitives are untouched.
- Size barely changes.

But this path is far harder than it looks, and the core obstacle is an unavoidable fact: **the subsetted original font simply does not contain the target simplified glyph**. The embedded font in Hong Kong stock reports is subsetted, containing only the traditional characters actually used; the simplified glyph "报" isn't in there at all. So the shortcut of "change the encoding to point to another glyph" doesn't work — we either modify the font, or switch that character to a newly introduced font resource.

And once we have to switch font resources, the troubles pile up layer by layer:

- We have to re-lay out positioning. We must pull that character out of the TJ array on its own, insert `Tf` to switch fonts and `Tj` to draw it, then use `Td` / `TJ` offsets to reconnect the baseline.
- We also have to compensate for horizontal compression. The Chinese in Hong Kong stock tables is laid out with horizontal compression (measured median compression ratio 0.894 / 0.837), so after switching fonts we still have to use `Tz` or extra offsets to restore it.
- We have to write a content-stream tokenizer ourselves, and correctly handle string escaping, TJ arrays, inline images (BI/ID/EI), Form XObjects (text is often inside XObjects rather than in the page content stream), and XObjects shared across multiple pages.
- Errors are silent. A mistake doesn't raise an error; instead the layout quietly breaks — characters crammed together, spilling out of cells. This is much harder to spot than A's "order scrambled but all characters correct". And rendering necessarily changes, so A's pixel gate is directly disabled here.

To sum up B in one sentence: layout is naturally correct and size unchanged, but it requires hand-writing parsing and re-layout at the PDF-syntax level — high complexity, and errors are hard to detect.

## 4. Approach C: Only Swap the Glyph in the Font, Leaving the Content Stream Untouched

Following the "three-layer structure" one step further: what if we don't touch the content stream (it just says "draw glyph number 0x27F3 here"), and instead modify the font, the recipe book — replace the outline of glyph number 0x27F3 (報) in the embedded font with the outline of "报", and at the same time change the mapping of that CID in ToUnicode to U+62A5 (报) — what happens?

The result: on the page, the traditional character is displayed as simplified in place, while the content stream is never touched at all.

This brings a string of very nice properties:

- Order, position, spacing, horizontal compression, vertical text, and rotated text are all naturally preserved. Because this information all lives in the content stream, and the content stream wasn't touched.
- No redaction needed, so no accidental deletion of neighboring characters and no lost images — the gate failures of "lost images" in the 4 interim reports also disappear as a result.
- No new fonts, no new text objects, so size barely changes, and may even shrink slightly.
- The workload is bounded and measurable. In practice there are 1143 glyph outlines total, covering 99.98% of the characters to be rewritten.
- The mechanism is already proven. This time we've already validated the "modify embedded font + inject CMap" mechanism (`remap_font_by_gid` + `update_stream`), and it's the same underlying mechanism as the ToUnicode fix we did earlier, so it can be reused.

Of course it has its costs too:

- We have to do outline format conversion. Noto CJK uses CFF curves, while HYQiHei uses glyf quadratic curves, so we need cu2qu + TTGlyphPen for the conversion; we also have to align the em box, baseline, and advance width (fortunately CJK full-width characters all have a width of 1000, so this item is low risk).
- The glyph is replaced globally. Once a certain GID is replaced, all of its occurrences in the document become simplified. In this document that happens to be exactly what we want, because the mapping is a pure function; but for another document, if the traditional-to-simplified conversion gives the same glyph different simplified results in different contexts, C can't express it — so we must do conflict detection first.
- 14 characters can't be changed. They use MSungHK / MHeiHK, which are non-embedded fonts (0.02%); the glyphs aren't in the document, so they can't be changed.
- Rendered pixels necessarily change (which is what we want anyway), so A's pixel gate is disabled. Verification has to switch to "character multiset + per-character baseline coordinates unchanged + sampled visual inspection". But "baseline coordinates unchanged per character" is actually a stronger check than A's — since the content stream wasn't modified, the coordinates must match exactly, and one comparison tells you whether anything went wrong.

To sum up C in one sentence: layout is naturally perfect, size unchanged, risk controllable, but it requires font outline conversion and depends on the premise that "the mapping is a pure function".

## 5. Putting the Three Paths Side by Side

Lining up the key dimensions of the three approaches makes their strengths and weaknesses clear.

- Text-layer order: A breaks it; B correct; C correct.
- Position / spacing / compression: A needs morph for approximate restoration; B needs manual offset recalculation; C naturally unchanged.
- Vertical / rotated text: A has known pitfalls, now just skipped; B needs dedicated handling; C naturally supported.
- Image / vector graphic risk: A has it (lost images in practice); B none; C none.
- Size: A grows 8%~37%; B barely changes; C barely changes.
- Implementation complexity: A low (done); B high (PDF-syntax level); C medium (font-outline level).
- Main risk: A is scrambled order and redaction deletion; B is silent layout breakage; C is outline conversion precision and global replacement.
- Coverage: A 100%; B 100%; C 99.98% (excluding non-embedded fonts).

Looking at this comparison, a natural conclusion emerges: C is almost across-the-board superior in "layout correctness, size, risk", and its only weakness is that 0.02% of non-embedded fonts it can't reach. And B happens to fill exactly that gap — for non-embedded fonts, switching font resources was required anyway.

## 6. Why the Final Answer Is a "Combined Approach"

None of them is perfect on its own:

- A's order breakage and lost images are structural defects that can't be patched.
- Using B for everything means the complexity and silent-breakage risk are too high; it's not worth hand-writing content-stream re-layout for 99.98% of the characters.
- C can't cover that 0.02% of non-embedded fonts.

So we let each of them do what it's best at, forming a pipeline:

1. Traditional → simplified conversion: use OpenCC to figure out, by context, what each character should become (why by context is detailed in the next section).
2. GID mapping conflict detection: confirm "whether the same glyph, across all its occurrences in the document, points to the same simplified character". Only when this condition holds is approach C's "global replacement" safe. This step is the gate for whether C can be used; details in the next section.
3. Three-tier routing:
   - First priority ★ Approach C: for characters in embedded fonts where that GID has a unique target, modify the glyph outline, leave the content stream untouched. This is the workhorse, handling the vast majority.
   - Second priority Approach B: for characters where that GID's target is not unique (one traditional to many simplified), or the font is non-embedded, do per-position font-switching rewrite. This fills the gaps.
   - Third priority keep traditional and warn: for cases where context extraction fails, OpenCC can't determine the result, the font structure is abnormal, etc., we'd rather not convert than convert wrongly, and record it for manual review.
4. Unified verification: use "character multiset conservation + per-character baseline coordinates unchanged + sampled visual inspection" to jointly validate the results of the several branches.

The elegance of this combination is: **with one conflict detection, we send the vast majority of characters onto the path that's "naturally correct in layout and loses no images" (C), hand only the few unreachable or ambiguous characters to the complex but necessary B, and use one conservative fallback to hold the lower bound on correctness**. We get both C's layout correctness and size advantage, use B to fill the coverage gap, and casually shed A's chronic image-loss problem.

## 7. Approach C's Most Hidden Trap: Traditional-to-Simplified Is Not One-to-One

Earlier we repeatedly mentioned that C's weakness is "the glyph is replaced globally". This sounds abstract, but behind it hides a fact that's extremely easy to overlook and enough to make C fail:

**Converting traditional to simplified is not always a character-level one-to-one mapping; there are cases where "one traditional character corresponds to multiple simplified characters".**

That is, the same traditional character may need to become different simplified characters in different words. A few typical examples:

- "乾": in "乾燥 (dry), 乾净 (clean)" it becomes "干", but in "乾隆 (Qianlong)" it stays "乾" — cannot be converted.
- "發 / 髮": the "發" in "發展 (develop), 發生 (occur)" becomes "发", and the "髮" in "頭髮 (hair)" also becomes "发" — two traditional characters merging into one simplified.
- "後 / 后": the "後" in "後来 (later), 後面 (behind)" becomes "后", while the "后" in "皇后 (empress)" was already "后".
- "臺 / 台": the "臺" in "臺灣 (Taiwan), 臺北 (Taipei)" becomes "台", and the "台" in "台阶 (steps), 舞台 (stage)" is also "台".
- "鐘 / 鍾": "鐘表 (clock)" becomes "钟", while the "鍾" in "鍾姓 (surname Zhong), 鍾馗 (Zhong Kui)" also becomes "钟".

The most illustrative of these is "乾":

- 乾燥 → 干燥
- 乾净 → 干净
- 乾隆 → 乾隆

So "乾 → 干" is not an unconditionally valid character mapping; it depends on the word it's in. Fortunately OpenCC solves exactly this — it converts by dictionary and by context, so `convert("乾隆")` correctly keeps "乾隆" rather than naively replacing every "乾" with "干".

### Why This Is Fatal for Approach C

The problem is: **OpenCC gives answers based on "character + context", while approach C acts based on "GID"** — these two granularities don't match up.

Suppose a PDF contains both "乾燥" and "乾隆", and the font happens to have:

- GID 0x123 = 乾

If approach C directly does "replace the outline of GID 0x123 with '干'", the result is:

- 乾燥 → 干燥 ✓
- 乾隆 → 干隆 ✗

Because the same GID is replaced globally, it can't handle "convert here, don't convert there". This is exactly the true destructive power of the phrase "the glyph is replaced globally".

### So the Granularity of Conflict Detection Must Be (GID, Context)

This is why we emphasize: **don't just do `GID → Unicode`; do `GID → occurrence positions → context → OpenCC → target Unicode`**.

Concretely, for each GID:

1. Find all its occurrence positions in the PDF.
2. Extract the context (surrounding characters) at each occurrence.
3. Use OpenCC to determine each one's target character by context.
4. Aggregate these target characters and see whether there's only one.

For example, GID 0x123 (乾) might yield:

- page 1: 乾燥 → 干
- page 5: 乾隆 → 乾
- page 8: 乾燥 → 干

After aggregation the target set is {干, 乾}, with two results. So the verdict: **this GID cannot use approach C's global replacement**.

In other words, C's "pure function" premise needs redefining — it's not verifying that `GID → Unicode` is unique, but verifying:

> For all occurrences of a given GID, after OpenCC's context-based conversion, the target Unicode is exactly the same.

Only a GID that satisfies this condition can be safely promoted to `GID → new glyph` and handed to approach C. Characters with a unique target like "報 → 报", "發 → 发", "經 → 经", "濟 → 济" are C's home turf; while a character with a non-unique target like "乾" must fall back to B for per-position handling. This also explains why B is irreplaceable here — B rewrites character by character, position by position, and can naturally express "the same traditional character, converted here, not converted there".

### Capturing the Key Point in One Sentence

OpenCC and approach C solve two different problems, which must be kept separate:

- What OpenCC solves is: "What character should this specific context become?"
- Approach C's constraint is: "Can the same GID be globally changed to the same thing?"

The former is a linguistic problem, the latter an engineering constraint. What truly decides whether a character goes to C or B is the latter — **for all occurrences of this GID, is there only one target character after OpenCC**.

For exactly this reason, what's really worth focusing our statistics on is two "small sets": one is characters in non-embedded fonts, the other is characters whose GID target is not unique. Given the current document is 99.98% embedded with only 14 non-embedded characters, as long as the second set is also small enough, "C as the main path, B as fallback" is a fairly stable engineering solution.

## 8. Lessons We Can Extract From This Decision

Looking back, the reasoning behind this choice can actually be reused in many "modifying an existing format/data" scenarios:

- First figure out "what layers this thing is assembled from". Once PDF's three-layer structure is seen through, the question "which layer is cheapest to touch" answers itself — modifying the content stream (B), modifying the font (C), or starting fresh and pasting new (A) correspond to completely different cost curves.
- Verification capability is itself part of the approach. The reason A was reassuring for a while is that it could do a pixel gate; while C traded up to a stronger verification dimension — "since the content stream wasn't touched, baseline coordinates must be unchanged per character". When choosing an approach, don't just look at functionality; look at whether it can afford a hard check.
- Don't fixate on "one approach to rule them all". When the processing cost differs enormously between 99.98% and 0.02%, routing is the more economical engineering decision. The key is finding that cheap "workhorse approach", plus a "premise check" that can hold the boundary (here, the GID conflict detection).

In the end, this combination essentially lets each approach work only in the range where its cost is lowest and risk is smallest, then stitches them together with one unified verification.

---

If you found this article helpful, feel free to **like, bookmark, and follow**. I'll keep sharing more valuable content. Your support is my greatest motivation to create!
