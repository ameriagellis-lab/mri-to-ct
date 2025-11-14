# MRI to CT Batch Converter (Web Interface)

This project provides a web-based tool that allows users to upload an **entire folder of MRI images** and receive a **batch of converted CT scans**. The frontend is built with HTML/JavaScript, and it communicates with a backend `/convert` endpoint that processes multiple MRI files at once.

---

## 🚀 Features

- Upload **folders** containing multiple MRI files  
- Supports `.nii`, `.dcm`, and `.img` formats  
- Sends multiple files to the backend using `FormData`  
- Backend returns a **ZIP file** containing the converted CT scans  
- Clean, responsive HTML interface  
- Compatible with Flask, FastAPI, Node.js, or other server frameworks

---

## 📁 Project Structure

A typical project layout might look like this:

