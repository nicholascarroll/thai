Found it, and it's a real bug — in `thaiWordBounds`, not in your tags file. Let me confirm the mechanism by compiling the logic in isolation.Yes, it's your `ctags.c` — `thaiWordBounds`, not the tags file.

The culprit is the backward fallback:

```c
if (!isThaiWordAt(row, cx)) {
    if (cx > 0 && isThaiWordAt(row, prevCPStart(row, cx)))
        cx = prevCPStart(row, cx);
```

U+200B occupies three bytes but zero screen cells. So when `cx` sits on a separator, your block cursor is drawn over the *next* word — the one the separator precedes — while `cx` is logically still on the separator. The separator isn't a Thai word character, the fallback fires, and it walks *backwards* into the previous word. Visually the cursor is on word B; the lookup returns word A.

Reproduced with your own helpers and `thaiWordBounds` copied verbatim, on ธรรม‌ZWSP‌ทั้งหลาย from verse 1:

```
cx=12 (on the ZWSP)  current -> "ธรรม"        fixed -> "ทั้งหลาย"
cx=0                 current -> "ธรรม"        fixed -> "ธรรม"
cx=15                current -> "ทั้งหลาย"    fixed -> "ทั้งหลาย"
```

The fix — step forward over zero-width characters *before* considering the fallback, since a zero-width char always visually belongs to what follows it:

```c
static int isZeroWidth(uint32_t cp) {
	return cp == 0x200B || cp == 0x200C || cp == 0x200D || cp == 0xFEFF;
}
```

then at the top of `thaiWordBounds`, before the existing `if`:

```c
	/* A zero-width separator takes a byte offset but no screen cell,
	 * so a block cursor drawn on the following character can leave cx
	 * sitting on the separator.  Step past it before the backward
	 * fallback, or M-. looks up the preceding word. */
	while (cx < row->size && row->chars[cx] >= 0x80 &&
	       isZeroWidth(utf8Decode(row->chars, cx)))
		cx += utf8_nBytes(row->chars[cx]);
```

Both cursor-inside-a-word cases are unchanged, so the fallback still works as intended for a bar cursor parked just after a word.

Two caveats. I couldn't build emil itself, so this is tested in isolation with the real `utf8_nBytes`, `utf8_isCont` and `utf8Decode` lifted from `unicode.c` — the logic is faithful, but it hasn't run inside the editor. And I'm inferring that your cursor motion leaves `cx` on the separator rather than skipping it; if your `←`/`→` commands already step over zero-width characters, then the problem is upstream in motion rather than here, and the symptom would instead only show when arriving via search or a jump. Worth checking where `cx` actually lands before patching.

Incidentally, `isThaiWordCP` agrees exactly with the `istok` ranges in `n-thai-vocab.sh`, so the tags file and the lookup are in sync — that part is sound.
