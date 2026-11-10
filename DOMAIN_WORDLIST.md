# Domain wordlists from technical audio and video

## Goal

Create local, newline-delimited lists of terms that recur in technical podcasts and YouTube videos within a defined subject area. The first two areas are **frontend web design** and **backend application design**. A term may be one word (`sharding`) or a short phrase (`columnar database`, `cubic-bezier easing function`). The lists are vocabulary indexes, not summaries, transcripts, or quotations.

The MVP succeeds when a reader can open `wordlists/frontend_web_design.txt` or `wordlists/backend_application_design.txt` and see a reviewed, ranked set of terms actually supported by the source corpus. The examples above are targets to look for, not terms to insert without evidence.

## MVP scope and outputs

- Produce one UTF-8 `.txt` file per domain, with one canonical term per line. Order terms by the ranking below; break ties alphabetically. No headers, counts, or commentary in these files.
- Produce a small optional sidecar, `wordlists/<domain>.jsonl`, with aggregate metadata for each term: `term`, `rank`, `source_count`, `publisher_count`, `occurrences_per_10k_words`, and `variants`. This is straightforward to generate from the same counters, but the `.txt` files remain the only required consumer-facing output.
- Keep a separate local source manifest for provenance and reproducibility. It records source ID, domain, medium, publisher, episode/video URL, date, language, transcript origin, permission or license basis, and ingestion status. It contains no transcript text.
- Use existing, permitted transcripts for the first release. Audio transcription is an optional later ingestion path when the owner permits it and the audio can be obtained under the applicable terms.
- Limit the MVP to English. Use a separately labeled corpus and list for any later language rather than mixing languages in one ranking.

Suggested layout:

```text
wordlists/
  frontend_web_design.txt
  frontend_web_design.jsonl       # optional aggregate metadata
  backend_application_design.txt
  backend_application_design.jsonl
data/
  sources.csv                     # local provenance manifest; no transcript text
.work/transcripts/               # ignored working files, only if retention is permitted
```

For example, a plain-text list might contain `cubic-bezier easing function` on one line and `design token` on another. These examples illustrate the format, not predetermined results.

## Source selection and access

1. Define a short inclusion guide for each domain. Frontend web design covers layout, styling, interaction, browser behavior, and accessibility. Backend application design covers data storage, service architecture, reliability, and scaling. Label each source with one primary domain; exclude broad interviews with little relevant discussion.
2. Build a pilot of roughly 10 permitted transcripts per domain, then expand toward 20–30 per domain before treating the list as representative. Aim for at least three independent publishers per domain and both podcasts and videos where permission allows. Record actual coverage alongside each release; do not claim broad coverage from a small or single-publisher sample.
3. Prefer transcripts supplied directly by creators or made available under a license that permits this processing. Podcast RSS may advertise a transcript through `<podcast:transcript>`; the tag is a discovery mechanism, so check the creator's stated terms or obtain permission before ingesting it. [Podcasting 2.0 transcript guide](https://podcasting2.org/docs/guides/how-to-add-transcripts-to-your-podcast)
4. For YouTube videos, ask the creator for a transcript file and permission to use it for aggregate vocabulary extraction. Do not build the MVP around scraping captions or downloading YouTube audio/video. The official captions download method requires permission to edit the video, and YouTube's API policies restrict aggregation and derived use of API data. A video URL in the manifest does not itself grant access to its captions. Reassess the applicable terms before adding any YouTube API integration. [YouTube captions download documentation](https://developers.google.com/youtube/data/reference/rest/v3/captions/download); [YouTube API developer policies](https://developers.google.com/youtube/terms/developer-policies)
5. Exclude duplicate uploads, cross-posted transcripts, trailers, and very short clips. Record transcript quality (`human`, `edited ASR`, or `raw ASR`) and prefer human or checked transcripts. If a source is unusable, record why and replace it with another source rather than silently counting it.

This access review is a gate for each source. Publishing only isolated terms reduces reproduction risk but does not by itself grant a right to collect or process the underlying material.

## Extraction and ranking pipeline

1. **Ingest:** Read permitted plain text, VTT, or SRT files from the local work area. Strip timestamps, cue numbers, speaker labels, and markup. Keep the manifest's source ID attached to the resulting token counts, but keep raw transcript content out of committed outputs.
2. **Normalize:** Apply Unicode normalization and case folding for counting; preserve conventional display forms for acronyms and product names. Join obvious caption line breaks, remove filler and generic stopwords, and normalize spelling variants such as `cubic bezier`/`cubic-bezier`. Do not indiscriminately stem technical words: `cache`, `caching`, and `cache invalidation` can have distinct meanings.
3. **Generate candidates:** Count single words and short phrases (two to four words) using token n-grams and simple phrase boundaries. Keep meaningful hyphens and punctuation inside terms, such as `cubic-bezier` and `CI/CD`. Reject sentence fragments, names of speakers or shows, ads, and generic conversational phrases. Do not use transcript sentences as examples or definitions.
4. **Count across sources:** For each candidate, track total occurrences, words analyzed, distinct podcast episodes or videos, and distinct publishers. Compute `occurrences_per_10k_words = 10,000 × occurrences / words_analyzed`. Deduplicate repeated captions within a source and cap any single source's contribution to ranking so a repeated slogan or a long interview cannot dominate.
5. **Find domain terms:** Prefer candidates found in at least three sources and two publishers in a domain. Rank primarily by the share of sources containing the term and its normalized frequency; use its frequency in the other domain or a general technical reference corpus to down-rank broad words such as `system` or `design`. Treat domain distinctiveness as a filter or modest ranking signal, since useful terms can occur in several areas.
6. **Review:** A domain-aware reviewer checks the top candidates for transcription errors, ambiguous phrases, misleading merges, and missing multiword forms. Merge spelling variants under one canonical term; retain a phrase when it has a distinct technical meaning (`columnar database` versus `database`). The reviewer may remove bad candidates but must not add a term that was absent from the permitted corpus.
7. **Export:** Write the ranked `.txt` list and, if enabled, the JSONL sidecar. Keep the code, domain definitions, stoplist, normalization rules, and generation settings under version control. Keep transcript files outside version control and delete them after processing unless retention is explicitly permitted and useful for an authorized rebuild.

The first implementation can be a small local script with deterministic input and output paths. No database, hosted service, browser interface, or machine learning model is needed for the MVP. Add automated speech recognition or more sophisticated phrase extraction only if the reviewed pilot shows that transcript availability or candidate quality is the limiting factor.

## Publication rules and quality checks

- Publish terms and aggregate counts only. Do not publish transcript passages, captions, timestamps, episode summaries, or source-specific word sequences. If descriptions are added later, write and review them independently rather than copying source wording.
- Inspect the top 50 terms in each domain manually. Sample another 20 terms from lower ranks to estimate how much noise survives the filters. Fix normalization or source quality issues and rerun before release.
- Check that each published term occurs in the corpus, duplicates collapse to one line, and the file is valid UTF-8 with no blank lines. Compare adjacent releases to catch sudden changes caused by one new source or a parsing error.
- Record the corpus size, source mix, date range, language, ranking method version, and review date in a small release note or README. Explain that frequency reflects the selected corpus, not the whole technical field.
- If a source permission changes or a creator requests removal, remove that source from the manifest's active set and regenerate the lists. Keep a record of which release used which approved source IDs.

## Delivery sequence

1. Write the two domain inclusion guides and source manifest schema; assemble and permission-check a small mixed-medium pilot.
2. Implement text/VTT/SRT parsing, candidate extraction, per-source counting, ranking, and deterministic file export.
3. Review pilot terms with someone familiar with each domain; adjust phrase rules, stopwords, and source balance.
4. Expand the permitted corpus toward the target size, regenerate both lists, and run the publication checks.
5. Release the text files, optional aggregate JSONL, and a short corpus/method note. Re-run the same process when enough new approved sources have accumulated to justify an update.

The MVP is complete when both domain files exist, each published term is traceable to permitted source counts, a reviewer has checked the candidate quality, and no source transcript text is included in the published artifacts.
