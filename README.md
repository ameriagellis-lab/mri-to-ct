# MRI to CT Converter

A small web UI for uploading an MRI file and converting it to a CT scan image. This README explains the intent of the page, how to run it locally, and includes the original HTML so you can copy it to an `index.html` file.

## What this is

- A simple HTML form that lets users pick an MRI file (.nii, .dcm, .img) and submit it to a backend endpoint at `/convert`.
- The page sends the file via `fetch` to `/convert` (POST, multipart/form-data) and expects a binary image blob in response (for example, a PNG).
- After receiving the blob, the page creates a downloadable link for the converted CT scan.

## Supported input file types

- .nii
- .dcm
- .img

## How to use

1. Provide an HTTP server/backend that listens for POST `/convert` and accepts `multipart/form-data` with the `mri` file field. The backend should return the converted CT image as a binary blob (e.g., `image/png`).
2. Save the HTML below as `index.html` (or include it in your app).
3. Serve the file (e.g., `python -m http.server`), or open in a browser while ensuring your backend is reachable at `/convert`.
   - Note: If serving the frontend separately from the backend, configure CORS on the backend or update the fetch URL to the backend's absolute URL.

## HTML (copy this into `index.html`)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>MRI to CT Converter</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 2em auto;
            max-width: 500px;
            padding: 1em;
            background: #f8f8f8;
            border-radius: 8px;
        }
        h1 { text-align: center; }
        .form-group { margin-bottom: 1em; }
        label { display: block; margin-bottom: 0.5em; }
        input[type="file"] { margin-bottom: 1em; }
        button { padding: 0.6em 1.5em; font-size: 1em; }
        .result, .error { margin-top: 1em; }
        .result a { color: #0077cc; }
    </style>
</head>
<body>
    <h1>MRI to CT Converter</h1>
    <form id="uploadForm" enctype="multipart/form-data">
        <div class="form-group">
            <label for="mri">Choose an MRI file to upload:</label>
            <input type="file" id="mri" name="mri" accept=".nii,.dcm,.img" required>
        </div>
        <button type="submit">Convert</button>
    </form>
    <div id="status"></div>
    <div class="result" id="result"></div>
    <div class="error" id="error"></div>

    <script>
        document.getElementById('uploadForm').onsubmit = async function(event) {
            event.preventDefault();
            document.getElementById('status').textContent = 'Uploading and converting, please wait...';
            document.getElementById('result').textContent = '';
            document.getElementById('error').textContent = '';
            let formData = new FormData(this);

            try {
                let res = await fetch('/convert', {method: 'POST', body: formData});
                if (!res.ok) {
                    throw new Error('Conversion failed.');
                }
                const blob = await res.blob();
                const url = URL.createObjectURL(blob);
                document.getElementById('status').textContent = '';
                document.getElementById('result').innerHTML = `
                    <a href="${url}" download="ct_scan.png">
                        ⬇️ Download Converted CT Scan
                    </a>
                `;
            } catch (err) {
                document.getElementById('status').textContent = '';
                document.getElementById('error').textContent = err.message;
            }
        };
    </script>
</body>
</html>
```

What's next: if you want, I can (1) produce a minimal example backend in Python (Flask) that accepts the upload and returns a placeholder image so you can test the frontend, or (2) convert this README into a more detailed developer guide with CORS examples and Docker instructions. Which would you like?
