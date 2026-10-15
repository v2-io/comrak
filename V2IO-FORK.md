# The v2-io fork of comrak

*For anyone landing in https://github.com/v2-io/comrak: what this fork changes, and what it may change next. Started 2026-10-15.*

This fork carries fixes and features that v2-io's tools need, starting with md-press, a Markdown canonicalizer that depends on comrak's source positions. Its patches are written with an LLM (Claude), so they are not offered upstream: comrak's [CONTRIBUTING.md](CONTRIBUTING.md) does not accept LLM-written code. Upstream is welcome to take anything from here. We won't send it.

## Branches

- `main` follows upstream (kivikakk/comrak).
- `sourcepos-fixes` is upstream `main` plus our patches. It's the branch to depend on:
  ```toml
  comrak = { git = "https://github.com/v2-io/comrak", branch = "sourcepos-fixes" }
  ```

## Landed

Three fixes to inline source positions. Each commit message gives the root cause:

- **Paragraph starting with definitions.** Inlines in a paragraph (or setext heading) that starts with link reference definitions came out too early by the number of definition lines, with the wrong line's indentation. This also covers a table split off after definitions.
- **Line break in a link's tail.** A line break between a link's text and its closing `)` or `]` made everything after it drift. That includes a break inside a multi-line title or a full reference's label. The link now ends at its real `)`. This replaces the end-at-destination position upstream settled on in #572.
- **Attributes spanning lines.** The same drift after an attribute block (`{…}`) that spans lines.

A security review of these found no vulnerabilities and no panics across about 10M fuzzed parses. It found two narrow position regressions, both of which come from older bugs in wikilinks and inline footnotes (listed below). The review is kept with the udon project: `v2/spikes/more-markdown/comrak-sourcepos-security-review.md` in https://github.com/v2-io/udon.

## Planned, not yet started: keeping reference links in the tree

So that Markdown can round-trip. Today `[text][label]`, `[text][]` and `[text]` all become a plain `Link { url, title }`; the label and the form are gone. Link reference definitions leave no node at all, and `format_link` in `src/cm.rs` writes every link inline.

**1. Record each link's form. This can be fully backwards compatible.**
- **Where:** the four forms are already separate branches in `handle_close_bracket` (`src/parser/inlines.rs`). The label is in hand there as written, before it's normalized.
- **Storage:** a new optional field on `Ast`, behind a cargo feature, the way `attrs` is. `Ast` already has private fields, so outside code can't build one from a struct literal or destructure it without `..`, and a new public field can't break anyone. The feature gate also keeps the 88-byte size assertion untouched, since that assertion only applies without the `attributes` feature.
- **Avoid** adding a field to `NodeLink`. That would break code that writes `NodeLink { url, title }`.

**2. Keep definitions, in one of two ways:**
- **(a) Fully backwards compatible:** keep definitions out of the tree. Collect each definition's label, URL, title and exact position into a field on the document node's `Ast`, or return them from a new function next to `parse_document`. Existing code can't tell the difference.
- **(b) The cleaner tree:** a new `NodeValue` variant for definitions, produced only under a parse option. The trees people get by default stay the same. But `NodeValue` isn't `#[non_exhaustive]`, so the new variant breaks, at compile time, any downstream `match` without a catch-all arm. comrak adds variants in minor releases, so this kind of break is normal for it, but it is a break.
- **Where:** after the definitions fix, `resolve_reference_link_definitions` (`src/parser/mod.rs`) knows each definition's exact lines (`removed_lines`) and prefix widths. That is the place to build either form. A paragraph that held only definitions, which is dropped today, would become those definitions.

**3. Write them back.** With 1 and 2, `src/cm.rs` can write `[text][label]`, `[text][]` and `[text]` and put the definitions back where they were. Two of its current rewrites would stop as well:
- an inline link whose text equals its URL is written back as `<url>`;
- a bare `www.` autolink is written back as `[www.…](http://www.…)`.

Until this exists, v2-io's `comrak-ast --links` tool (udon repo, `v2/spikes/more-markdown/comrak-ast/`) reads the link forms back from source positions instead.

## Known position bugs still open

From the security review. A tool that slices input by `sourcepos` should use checked slicing (`str::get`) regardless.

- **Wikilinks spanning lines.** A wikilink that spans a line break doesn't advance the line count, so everything after it is one line early. The fix goes in `handle_wikilink`, in the same way the link fix advances the line.
- **Inline footnotes spanning lines.** A multi-line inline footnote `^[…]` uses its own coordinates: line 1 is the footnote's own first line, and the offsets are into its own text. Text after it stays on its first line. The fix is to give `handle_inline_footnote` the document's line and per-line indentation.
- **Tabs in list items.** Block end columns on tab-indented continuation lines inside list items run past the line. A tab partly used up by list indentation can also put a column in the middle of a character.
- **Description-list terms with multi-byte text** can end in the middle of a character.
- **`sourcepos_chars` with lone `\r` line endings.** The columns are converted against the wrong line.
