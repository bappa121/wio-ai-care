# Wio AI Care System

A simple Flask-based medical care management system that allows:
- Citizen registration
- Medical data summarization
- Web-based interface for interaction

## 🌐 Live Demo
You can deploy this on [Render.com](https://render.com) for free. Instructions below.

---

## 🛠 Features

- Register citizens with unique UMN (Universal Medical Number)
- View medical summaries
- Web UI using Flask templates
- Ready for deployment on Render

---

## 📁 Project Structure

```
wio_app/
├── app.py               # Main Flask application
├── templates/           # HTML templates
│   ├── index.html
│   ├── register.html
│   └── summary.html
├── requirements.txt     # Python dependencies
├── Procfile             # For deployment on Render
└── render.yaml          # Optional auto-deploy config
```

---

## 🚀 Run Locally

1. Create a virtual environment and activate it:
   ```bash
   python -m venv venv
   source venv/bin/activate   # Windows: venv\Scripts\activate
   ```

2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

3. Run the app:
   ```bash
   python app.py
   ```

4. Visit `http://localhost:5000` in your browser.

---

## ☁ Deploy to Render

1. Push this repo to GitHub
2. Go to [https://render.com](https://render.com)
3. Click **“New Web Service”**
4. Use these settings:
   - **Environment:** Python
   - **Build Command:** `pip install -r requirements.txt`
   - **Start Command:** `gunicorn app:app`
   - **Free Plan**

Your app will be live at `https://your-app-name.onrender.com`

---

## 📬 Contact

Created by [Mashiqur R. Fahim](https://github.com/yourusername)  
Feel free to contribute or open issues!

