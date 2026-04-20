# olmocr patch: drop failed-page pdftotext fallback

## What

A small local modification to olmocr's pipeline that changes how pages which
fail all OCR retries are handled.

- **Before**: when a page exhausted `--max_page_retries`, olmocr silently
  substituted the page's `pdftotext` output (the PDF's embedded text layer)
  into the final markdown. For rotated or scanned pages without a clean text
  layer, this injected garbage / character soup.
- **After**: failed pages contribute **no text** to the markdown. They are
  still tracked internally (`is_fallback=True`, counted against
  `--max_page_error_rate`, reported in Dolma metadata), but nothing from the
  page lands in the output file.

Everything else is unchanged:

- Pages that succeed on any retry are written normally.
- The whole-document discard check (`num_fallback_pages / num_pages >
  --max_page_error_rate`) still runs. Pair this patch with
  `--max_page_error_rate 1.0` if you never want a document to be discarded.
- Metadata arrays (`primary_language`, `is_table`, `pdf_page_numbers`, etc.)
  keep their original length and positional alignment with physical page
  numbers.

## Where

File patched:

```
/home/<user>/miniconda3/envs/olmocr/lib/python3.11/site-packages/olmocr/pipeline.py
```

Backup of the original (pre-patch) file:

```
/home/<user>/miniconda3/envs/olmocr/lib/python3.11/site-packages/olmocr/pipeline.py.bak
```

Function modified: `make_fallback_result(...)`.

## The change

Single-line logic change inside `make_fallback_result`:

```diff
-            natural_text=get_anchor_text(pdf_local_path, page_num, pdf_engine="pdftotext"),
+            natural_text=None,
```

The docstring was also expanded to explain the new behavior.

## Verifying the patch is applied

```bash
grep -n "natural_text=None" \
  /home/<user>/miniconda3/envs/olmocr/lib/python3.11/site-packages/olmocr/pipeline.py
```

Expect a match inside the `make_fallback_result` function (around line ~246).

Or view the diff against the backup:

```bash
diff \
  /home/<user>/miniconda3/envs/olmocr/lib/python3.11/site-packages/olmocr/pipeline.py.bak \
  /home/<user>/miniconda3/envs/olmocr/lib/python3.11/site-packages/olmocr/pipeline.py
```

## Rolling back

```bash
cp \
  /home/<user>/miniconda3/envs/olmocr/lib/python3.11/site-packages/olmocr/pipeline.py.bak \
  /home/<user>/miniconda3/envs/olmocr/lib/python3.11/site-packages/olmocr/pipeline.py
```

## Re-applying after an olmocr upgrade

`pip install -U olmocr` (or rebuilding the conda env) will overwrite
`pipeline.py` and the patch will be lost. To reapply, either run the `diff`
above as a patch, or re-edit the file and change:

```python
natural_text=get_anchor_text(pdf_local_path, page_num, pdf_engine="pdftotext"),
```

inside `make_fallback_result` to:

```python
natural_text=None,
```

## Recommended CLI flags with this patch

To guarantee no document is ever discarded because of failed pages
(only the failed pages themselves are omitted from the markdown):

```
--max_page_error_rate 1.0
```

Optional quality knobs that reduce the chance of pages failing in the first
place:

```
--max_page_retries 12
--guided_decoding
--target_longest_image_dim 1280
```
