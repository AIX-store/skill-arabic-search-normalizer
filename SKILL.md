---
name: arabic-search-normalizer
description: Designs Arabic-aware search for apps and databases. Defines a normalisation pipeline (alef and hamza forms, taa marbuta and haa, alef maqsura and yaa, tatweel, diacritics, Arabic-Indic and Persian digits, Persian letter variants), weighs light stemming against root stemming, plans indexing and ranking, and produces a test table. Gives database and search-engine configuration advice in prose. Use when Arabic search misses obvious matches, returns noise, or when building search, autocomplete or deduplication over Arabic names and text.
---

# Arabic Search Normaliser

Make a user typing "احمد" find "أحمد", "أَحْمَد" and "احـــمد" — without making "مدرسة" match
everything that shares a root.

## When to use
- Search, filters or autocomplete over Arabic product names, people, places or documents.
- Customer or company deduplication where names are typed inconsistently.
- Complaints like "search works in English but not Arabic".

## 1. Normalisation pipeline (apply identically at index time and query time)
Apply in this order and document every step, because any asymmetry breaks matching:
1. **Unicode normalisation** to NFKC (folds presentation forms and compatibility glyphs).
2. **Remove tatweel** (ـ, U+0640).
3. **Remove diacritics** (harakat U+064B–U+0652, superscript alef U+0670, Quranic marks).
4. **Alef forms → bare alef:** أ إ آ ٱ → ا.
5. **Hamza on carriers:** ؤ → و and ئ → ي (optional; improves recall for misspellings,
   loses some precision). Standalone ء: keep or drop by test results.
6. **Alef maqsura → yaa:** ى → ي (users type them interchangeably, especially in Egypt).
7. **Taa marbuta → haa:** ة → ه (the most common end-of-word spelling variation).
8. **Persian/Urdu letter variants:** ک → ك, ی → ي, ہ → ه (keyboards and copied text).
9. **Digits:** Arabic-Indic ٠–٩ and Eastern Arabic-Indic ۰–۹ → 0–9.
10. **Lowercase and trim Latin**, collapse whitespace, strip zero-width characters and bidi
    control marks.
Keep the original text for display; normalised text is for matching only.

## 2. Stemming trade-offs
- **No stemming:** precise, poor recall on prefixes (ال، و، ب، ل، ف) and suffixes.
- **Light stemming** (strip common prefixes such as ال، وال، بال، كال، فال، لل and suffixes such
  as ها، ان، ات، ون، ين، يه، ة): best default for product and document search.
- **Root stemming** (reduce to the three-letter root): high recall, poor precision —
  كتاب، مكتب، كاتب all collapse together. Use only for exploratory research search.
- **Names:** never stem personal or company names; normalise only. Add a separate
  transliteration field (Mohammed / Muhammad / Mohamed) if Latin queries must find Arabic
  names.

## 3. Index design
- Two fields per searchable text: `exact_normalised` (pipeline only) and `stemmed`
  (pipeline + light stem). Boost exact matches above stemmed ones.
- Add a prefix or edge-n-gram field for autocomplete; Arabic words are short after
  stripping "ال", so start n-grams at 2 characters.
- For fuzzy matching, allow edit distance 1 for words of 4+ letters; Arabic words are short
  and distance 2 causes noise.
- Treat the leading "ال" as optional at query time even without stemming.

## 4. Engine configuration (in prose)
- **PostgreSQL:** create an immutable normalisation function implementing the pipeline,
  index its output with a trigram index for fuzzy and prefix matching, and for full-text
  search use the `simple` configuration over normalised text (there is no strong built-in
  Arabic stemmer); add light stemming in the function if needed. Unaccent alone does not
  cover Arabic letter variants.
- **Elasticsearch / OpenSearch:** the built-in Arabic analyzer includes a normaliser and a
  light stemmer; add a character filter for tatweel, Persian variants and digits, and a
  separate sub-field without stemming for exact and name matching.
- **SQLite FTS / MySQL:** normalise in application code before writing and before querying;
  their tokenisers do not fold Arabic variants.
- **Vector/semantic search:** still normalise input; embeddings are sensitive to tatweel,
  diacritics and digit scripts.

## 5. Test table (build before tuning)

| Query | Should match | Should NOT match | Rule exercised |
|---|---|---|---|
| احمد | أحمد، إحمد، أَحْمَد | — | alef + diacritics |
| مدرسه | مدرسة، المدرسة | مدرس | taa marbuta, ال |
| مستشفى | مستشفي | — | alef maqsura |
| كتــاب | كتاب | كاتب (unless root mode) | tatweel, no root stem |
| ٢٠٢٦ | 2026 | — | digits |
| مصطفي | مصطفى | — | alef maqsura (Egyptian typing) |
| شركه النور | شركة النور | شركة النورس (exact field) | multi-word + taa marbuta |

Measure recall and precision on 50–200 real queries from logs before and after each rule.

## Output format
Return: the pipeline as an ordered list with the decision for each optional step, the field
and analyzer plan, engine-specific notes, and the test table extended with domain queries.

## Pitfalls
- Normalising only the query or only the index.
- Root stemming product catalogues and names.
- Folding ة→ه but displaying the normalised form to users.
- Ignoring zero-width joiners and RTL marks pasted from documents.
- Highlighting matches by offsets in normalised text that no longer align with display text.
