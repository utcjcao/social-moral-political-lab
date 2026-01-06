# Current issues

The data processing and training done in this repository can be significantly improved for speed and memory usage. Below are some notes on what can be improved.

**step1_combine_json**

- The Dropbox path processing loads all records into memory before writing the CSV. Stream rows directly to disk to avoid memory blow-ups on large years.

**step2_lemmatize_data**

- Current flow processes one row at a time and accumulates all chunks before writing, which is extremely slow and defeats chunking. Stream each chunk directly to the output CSV.
- The text cleaning and lemmatization steps are split across multiple passes and sometimes re-tokenize data inconsistently. Use a single spaCy pipeline pass per chunk (normalize text -> nlp.pipe -> lemmatize + stopword filter).
- Parallel apply on very small chunk sizes creates heavy overhead and can be slower than serial processing. Increase chunk size and rely on spaCy's internal batching.
- The NLTK words filter can drop valid tokens (names, domain terms). Make it optional or replace with a lighter heuristic (alphabetic + length).
- Debug/legacy cells and undefined helper calls can cause runtime errors when run in order. Move them to a separate section or remove from the main workflow.

**step2_stem_data**

- The pipeline should stream chunks to the output instead of storing all processed chunks in memory.
- Avoid per-row processing with tiny chunks; use a larger chunk size and batch operations for speed.
- Consolidate text normalization and stemming into a single pass per chunk to reduce repeated work.
- Any debug or legacy cells should be separated from the main workflow to avoid accidental execution errors.
- Prefer configurable parameters (years, chunk size, output path) at the top for safer reruns.

**step3_calculate_honor**

- Counting relies on `article.split()` with no normalization; if `content` is stored as a list string, tokens will include brackets/quotes and skew counts.
- The script processes full years in-memory with `progress_apply`, which is slow at multi-million rows; chunked processing or vectorized approaches would reduce runtime.

**step3_calculate_threat**

- `dataset_folder` points to `processedNewsData`, but the code reads `labels.csv`, `emfd_amp.csv`, and `threat_corpus.txt` from it, which likely belong in the datasets folder and will fail in a fresh run.
- `get_overall_proportion` loops row-by-row in Python, which is very slow for large datasets; use vectorized tokenization or precomputed counts to scale.
- Several heavy dependencies/models (BERT) are defined but unused, adding install time and memory overhead without affecting outputs.

**step3_load_vectorizers_for_mf**

- `years` is overwritten with `['sample']`, so it will not process the intended years unless manually edited.
- `doc_strings` uses `str(row['content'])`, and `doc_generator` splits on the string representation of lists; this introduces brackets/quotes into tokens and can corrupt vectorization.
- The notebook loads entire CSVs into memory and iterates with `iterrows`, which is slow and memory heavy; use chunked reads or faster column operations.

**step4_train_models_gensim_for_mf**

- The GoogleNews word2vec model is loaded every run without a cache or limit and is extremely slow/heavy; consider a smaller model or keyed subset and persist the loaded model.
- There are duplicate definitions and execution order pitfalls (`open_dict` is defined twice and the main loop runs before some definitions), which can trigger NameErrors depending on cell order.

**step5_multilevel_testing**

- The notebook depends on interactive package installs and OAuth prompts (googledrive) every run, which makes it non-reproducible and breaks non-interactive execution.
- File paths are hard-coded and include a trailing space (`test1_Mind and Body `) that leads to file-not-found errors.
- Several cells assume objects are already defined (`all_data`, `merged_df`, `data`) and will error when run top-to-bottom without manual setup.
