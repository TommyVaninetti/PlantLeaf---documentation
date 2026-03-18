# 🌐 PlantLeaf — Website Database

**Version:** 1.0
**Last Updated:** March 18, 2026
**Author:** Frida Tirari

---

The PlantLeaf Website and Database platform is a full-stack web system designed to make experimental plant bioelectric and bioacoustic data publicly accessible, explorable, and downloadable.
The platform integrates a structured frontend, a dynamic backend API, and a lightweight database system to support open scientific research and data sharing.
It is part of the PlantLeaf project, enabling transparent access to experimental datasets and technical documentation related to plant signal acquisition.

---

## 🎯 Purpose and Mission

The PlantLeaf website aims to make research on plant bioelectrical and acoustic emissions accessible to a wider audience.
Plant communication science is still an emerging field, often limited by:

- high instrumentation costs
- restricted access to experimental data
- complex technical barriers

PlantLeaf addresses these challenges by providing:

- an open-access platform
- a structured and intuitive interface
- freely downloadable experimental datasets

The platform is designed for:

- researchers
- students
- educators
- curious users

It combines scientific rigor with accessibility, guiding users from biological concepts to technical implementation details.

---

## 🌐 Website Structure

The website is composed of seven main pages, each designed with a specific role.

### 🏠 Home — index.html

The homepage acts as the main entry point and includes:

- a mission overview
- a navigation grid with four interactive cards:
  - Voltage
  - Audio
  - Application
  - Database

### ⚡ Voltage — voltage.html

This page documents plant bioelectrical signals, including:

- acquisition methodology
- hardware setup
- signal analysis
- experimental results

Content includes graphs, images, and technical explanations.

### 🔊 Audio — audio.html

Dedicated to ultrasonic emissions in plants, this section includes:

- acquisition system description
- signal properties
- FFT-based analysis
- experimental conclusions

### 💻 Application — application.html

Describes the desktop software used for signal analysis, including:

- FFT Spectrum View
- Microphone Normalization
- Inverse FFT
- Envelope & Decay Analysis
- ultrasonic click detection algorithm

### 🗄️ Database — database.html

The public interface for experimental data access.

Features:

- dynamic table of files
- real-time API loading
- metadata visualization
- download tracking

### 👥 Team — team.html

Displays team members with:

- profile images
- roles
- contact emails

### ⚙️ Tech — tech.html

A technical page currently under development, intended for future expansions.

---

## 🎨 Design System — CSS Architecture

The entire visual system is defined in `styles.css`.

### 🎨 Visual Identity

- **Background:** `rgb(10,10,10)` (near black)
- **Primary accent:** `rgb(46,125,50)` (green)
- **Text:** white for high contrast

### 🧭 Navigation

- Fixed top navbar (`position: fixed`)
- Frosted glass effect (`backdrop-filter: blur`)
- Smooth hover transitions

---

## ⚙️ JavaScript — app.js

The shared JavaScript file manages three core features.

### 1️⃣ Smooth Scroll

- intercepts anchor links
- uses `scrollIntoView({ behavior: "smooth" })`

### 2️⃣ Parallax Effect

- applied to homepage hero
- based on `window.pageYOffset`
- includes opacity fade-out

### 3️⃣ Fade-in Animations

- implemented via `IntersectionObserver`
- triggers at 20% visibility
- uses opacity + transform transitions

---

## 🗄️ Database — System Architecture

The database system consists of:

- frontend (`database.html`)
- backend API (Flask)
- CLI upload script
- SQLite database

**Database file:** `plantleaf.db`

### 📊 Table: files

| Field | Type | Description |
|---|---|---|
| id | INTEGER | primary key (auto-increment) |
| filename | TEXT | sanitized file name |
| file_type | TEXT | category (audio, voltage, excel, csv) |
| description | TEXT | optional text |
| upload_date | TEXT | YYYY-MM-DD |
| downloads | INTEGER | integer counter, default 0 |

---

## ⚙️ Backend — app.py

The backend is built with Flask.

### Core Features

- CORS enabled via Flask-CORS
- automatic database initialization
- upload folder management

### 🔐 File Validation

Function: `allowed_file()`

Checks:
- valid extension
- case-insensitive matching

Allowed extensions:
- `.paudio`
- `.pvolt`
- `.xlsx`
- `.csv`

### 🔌 API Endpoints

#### `GET /api/files`
Returns all files sorted by newest.

#### `POST /api/upload`
Handles file upload:
- validates input
- saves file
- inserts database record

#### `GET /api/download/<filename>`
- verifies file existence
- increments download counter
- sends file as attachment

#### `DELETE /api/delete/<filename>`
- removes file from disk
- deletes database entry

### 🌍 Static Routing

- `/` → `index.html`
- `/<path>` → static files

---

## 📄 Frontend Database — database.html

Handles API interaction via inline JavaScript.

### 🔄 loadFiles()

- fetches `/api/files`
- dynamically builds table rows
- maps file types to colored UI badges

### ⬇️ downloadFile()

- redirects to download endpoint
- refreshes table after download

### 📅 formatDate()

Converts date format:
```
YYYY-MM-DD → DD/MM/YYYY
```

---

## 🌍 Deployment

The platform is deployed using Render.com.

**Configuration:**
- defined in `render.yaml`
- includes environment setup and start command

**Public URL:** [https://plantleaf.it](https://plantleaf.it)

---

## 📦 Requirements & Installation

**Dependencies:**
```
Flask==3.0.0
Flask-CORS==4.0.0
Werkzeug==3.0.1
```

Install with:
```bash
pip install -r requirements.txt
```

Start the server locally:
```bash
python app.py
```

Server available at: `http://localhost:5000`

---

## 👩‍💻 Development

**Website & Database**
Frida Tirari

Developed as part of the PlantLeaf research project.

**Last Updated:** March 18, 2026
**Version:** 1.0
