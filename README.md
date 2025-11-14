# MRI to CT Batch Converter (Web Interface)

This repository provides a simple web interface to upload a folder or multiple files (DICOM, NIfTI, PNG, PDF, etc.) and receive a ZIP file containing converted CT scans. The project is intentionally backend-agnostic — examples for Flask (Python) and Node/Express (JavaScript) are included to show how your server should accept folder uploads and return a ZIP.

This README describes:
- How the frontend uploads folders/files (including webkitdirectory folder upload)
- The expected backend endpoint behavior
- Example backend implementations (Flask and Node/Express)
- Supported file types and security notes
- Example usage and troubleshooting

---

## Key idea

- Frontend: sends a FormData object with one field name (e.g. `files`) that contains all files selected from a folder (using `input type="file" webkitdirectory` or drag/drop).
- Backend: reads `request.files` (or equivalent), processes/convert each file to CT, builds a ZIP archive, and returns it as `application/zip` with `Content-Disposition: attachment`.
- The frontend then triggers a download of the returned ZIP.

---

## Browser / Frontend notes

- Folder uploads in browsers are done using the nonstandard `webkitdirectory` (supported in Chromium-based browsers and recent Safari). Firefox supports `directory` in some contexts; behavior varies by browser.
- If the browser doesn't support folder upload, provide a drag-and-drop or multi-file input fallback so users can select many files manually.

Example HTML + JavaScript (drop this into your frontend HTML):

```html
<!-- input that lets users pick a folder (Chrome/Edge/Safari) -->
<input id="folderInput" type="file" webkitdirectory multiple />

<button id="uploadBtn">Upload & Convert</button>

<script>
document.getElementById('uploadBtn').addEventListener('click', async () => {
  const input = document.getElementById('folderInput');
  if (!input.files || input.files.length === 0) {
    alert('Please select a folder or files first.');
    return;
  }

  const form = new FormData();
  // Append all files. Use webkitRelativePath when available to preserve folder paths inside the zip
  for (const file of input.files) {
    // You can use a single field name 'files' repeated (many servers handle this)
    // or 'files[]' — ensure backend expects the same field name.
    form.append('files', file, file.webkitRelativePath || file.name);
  }

  try {
    const resp = await fetch('/api/convert', {
      method: 'POST',
      body: form
      // If using CORS to a separate backend, make sure to handle credentials and CORS headers
    });

    if (!resp.ok) {
      const text = await resp.text();
      throw new Error(`Upload failed: ${resp.status} ${text}`);
    }

    // The backend returns a zip file
    const blob = await resp.blob();
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'converted_cts.zip';
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
  } catch (err) {
    console.error(err);
    alert('Upload or conversion failed: ' + err.message);
  }
});
</script>
```

Notes:
- We append each file under the same field name `files`. On the backend, call `getlist('files')` (Flask) or `req.files` (Node) to get all submitted files.
- Using `file.webkitRelativePath` when present preserves original folder structure inside the ZIP.

---

## Supported file types

Typical MRI / image types you may accept:
- DICOM (.dcm, usually multipart directories)
- NIfTI (.nii, .nii.gz)
- Analyze (.img/.hdr)
- PNG / JPG / JPEG / TIFF - for single-slice images
- PDF — if you intentionally want to accept reporting PDFs, but conversion to CT likely not relevant
- Others: .mha, .mhd (if you support them)

Configure an allowlist on the backend (validate extensions and MIME types). Also apply file size limits.

---

## Expected backend endpoint

- Path: POST /api/convert (example)
- Request: multipart/form-data, many files with field name `files` (or `files[]`)
- Response on success: application/zip, Content-Disposition attachment; file name like `converted_cts.zip`
- Response on failure: HTTP 4xx/5xx with explanation JSON or text.

---

## Example backend (Flask) — minimal skeleton

This example:
- Accepts multiple files
- Saves them to a temporary directory
- Calls a placeholder `convert_to_ct` function (you must implement conversion)
- Zips converted outputs in-memory and returns as download

```python
# app.py (Flask example)
from flask import Flask, request, send_file, jsonify
import tempfile
import os
import io
import zipfile

app = Flask(__name__)
# configure e.g., app.config['MAX_CONTENT_LENGTH'] = 2 * 1024 * 1024 * 1024  # 2GB

ALLOWED_EXT = {'.dcm', '.nii', '.nii.gz', '.img', '.hdr', '.png', '.jpg', '.jpeg', '.tiff', '.pdf'}

def allowed(filename):
    lower = filename.lower()
    return any(lower.endswith(ext) for ext in ALLOWED_EXT)

def convert_to_ct(input_path, output_path):
    """
    Placeholder: implement your MRI->CT conversion here.
    For now just copy or produce a dummy file.
    """
    # Example: create a text file representing converted CT
    with open(output_path, 'wb') as f_out:
        f_out.write(b'CT data for ' + os.path.basename(input_path).encode('utf-8'))

@app.route('/api/convert', methods=['POST'])
def convert():
    files = request.files.getlist('files')
    if not files:
        return jsonify({'error': 'No files uploaded'}), 400

    # Create temp dirs
    with tempfile.TemporaryDirectory() as in_dir, tempfile.TemporaryDirectory() as out_dir:
        saved_inputs = []
        for f in files:
            filename = f.filename or 'unnamed'
            if not allowed(filename):
                # skip or reject
                continue
            safe_path = os.path.join(in_dir, filename)
            os.makedirs(os.path.dirname(safe_path), exist_ok=True)
            f.save(safe_path)
            saved_inputs.append(safe_path)

        if not saved_inputs:
            return jsonify({'error': 'No allowed files uploaded'}), 400

        # Convert each input to CT (implement your conversion pipeline)
        outputs = []
        for input_path in saved_inputs:
            out_name = os.path.splitext(os.path.basename(input_path))[0] + '_ct.nii'
            out_path = os.path.join(out_dir, out_name)
            convert_to_ct(input_path, out_path)
            outputs.append((out_name, out_path))

        # Make zip in-memory
        zip_buffer = io.BytesIO()
        with zipfile.ZipFile(zip_buffer, 'w', zipfile.ZIP_DEFLATED) as zf:
            for out_name, out_path in outputs:
                zf.write(out_path, arcname=out_name)
        zip_buffer.seek(0)
        return send_file(zip_buffer, mimetype='application/zip', as_attachment=True, download_name='converted_cts.zip')
```

Important:
- Replace `convert_to_ct` with your actual conversion pipeline (nibabel, pydicom, specialized model inference, etc.).
- Ensure `MAX_CONTENT_LENGTH` and file validation are set to safe values.

---

## Example backend (Node / Express) — minimal skeleton

This Node example uses `multer` (in-memory) and `archiver` to create an in-memory zip.

```js
// server.js (Node/Express example)
const express = require('express');
const multer = require('multer');
const archiver = require('archiver');
const fs = require('fs').promises;
const path = require('path');

const upload = multer({ storage: multer.memoryStorage() }); // or disk storage
const app = express();

const ALLOWED_EXT = new Set(['.dcm', '.nii', '.nii.gz', '.img', '.hdr', '.png', '.jpg', '.jpeg', '.tiff', '.pdf']);

function allowed(filename) {
  const ext = path.extname(filename).toLowerCase();
  return ALLOWED_EXT.has(ext) || filename.toLowerCase().endsWith('.nii.gz');
}

// placeholder conversion - replace with real logic
async function convertToCt(buffer, filename) {
  // For demo, return the same buffer or create a new buffer
  return Buffer.from('CT data for ' + filename);
}

app.post('/api/convert', upload.array('files'), async (req, res) => {
  const files = req.files || [];
  if (!files.length) return res.status(400).send('No files uploaded');

  const archive = archiver('zip', { zlib: { level: 9 } });
  res.attachment('converted_cts.zip');
  archive.pipe(res);

  for (const f of files) {
    const name = f.originalname || 'unnamed';
    if (!allowed(name)) continue;
    // Do conversion (buffer -> buffer)
    const convertedBuf = await convertToCt(f.buffer, name);
    // Add to zip. If you want to preserve folder structure, use f.fieldname or other metadata
    archive.append(convertedBuf, { name: name.replace(/\.[^.]+$/, '') + '_ct.nii' });
  }

  archive.finalize().catch(err => {
    console.error('Archive error', err);
    if (!res.headersSent) res.status(500).send('Server error');
  });
});

app.listen(3000, () => console.log('Server listening on 3000'));
```

---

## Example curl

Upload individual files:

```bash
curl -X POST -F "files=@/path/to/file1.dcm" -F "files=@/path/to/file2.nii" http://localhost:5000/api/convert --output converted_cts.zip
```

Note: curl does not support directory uploads directly — collect files recursively and send them all, or use a client-side folder upload as shown in the HTML sample.

---

## Security & operational notes

- Validate file types and MIME types server-side — never trust client-provided file names.
- Limit file sizes and total request size (e.g., Flask `MAX_CONTENT_LENGTH`).
- Sanitize filenames before saving; prefer to generate server-side safe names.
- Use virus scanning if receiving arbitrary user uploads.
- Rate-limit and authenticate API if publicly accessible.
- Consider streaming large zips back instead of loading entire archive in memory.
- Preserve folder structure inside your ZIP by using `webkitRelativePath` provided by the browser when available.

---

## Troubleshooting

- If you get empty `request.files` on the server, ensure the frontend sends `multipart/form-data` and that the server does not reject the request due to size limits or CORS.
- If the zip is corrupt, ensure you finalize the zip stream properly (e.g. `archive.finalize()` for archiver or closing the ZipFile buffer in Python).
- If folder paths are missing inside the zip, confirm the frontend includes `file.webkitRelativePath` when appending to FormData and that backend uses that name for archiving.

---

## What you need to implement / next steps

- Implement the actual MRI → CT conversion pipeline:
  - NIfTI: nibabel, numpy
  - DICOM: pydicom for parsing; convert series to volume
  - Optionally, a machine-learning model to synthesize CT-like images from MR input
- Add integration tests and sample data
- Add UI improvements: progress UI, large upload chunking, server-side streaming progress events (SSE or websockets)

---

If you'd like, I can:
- Provide a ready-to-run minimal Flask repo with a working example and an example HTML page wired up to it, or
- Provide a Node/Express project scaffold with the same frontend wired.

Tell me which backend you prefer and whether you want a runnable example included in the repo (I can generate the files).
