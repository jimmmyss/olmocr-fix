# olmocr patch: drop failed-page pdftotext fallback

## What is it?

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

Function modified: `make_fallback_result(...)`.

## The change

Single-line logic change inside `make_fallback_result`:

```diff
-            natural_text=get_anchor_text(pdf_local_path, page_num, pdf_engine="pdftotext"),
+            natural_text=None,
```

The docstring was also expanded to explain the new behavior.

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

## Recommended end-to-end run

Build the PDF list and launch olmocr in the background. Adjust `<user>` and
the dataset name (`archetai` below) to your setup.

```bash
find /home/<user>/datasets/archetai/raw -maxdepth 1 -name "*.pdf" > /home/<user>/pdfs_archetai.txt
wc -l /home/<user>/pdfs_archetai.txt

nohup olmocr /home/<user>/datasets/archetai \
  --pdfs /home/<user>/pdfs_archetai.txt \
  --model allenai/olmOCR-2-7B-1025 \
  --data-parallel-size 16 \
  --tensor-parallel-size 1 \
  --gpu-memory-utilization 0.92 \
  --max_model_len 16384 \
  --max_page_error_rate 1.0
  --target_longest_image_dim 1024 \
  --pages_per_group 100 \
  --workers 128 \
  --markdown \
  > /home/<user>/olmocr_archetai.log 2>&1 &
```
