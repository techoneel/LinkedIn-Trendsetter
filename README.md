# LinkedIn-Trendsetter

Here's a **step-by-step guide** to build the **LinkedIn Post Optimizer Chrome Extension** with scraping + AI analysis:

---

### **Step 1: Setup & API Keys**
#### 1. Get API Keys:
- **LinkedIn API**: Use [LinkedIn Marketing Developer Platform](https://developer.linkedin.com/) (Request access)
- **OpenAI/Gemini**: Follow [previous guide](https://www.linkedin.com/pulse/step-by-step-guide-get-keys-use-chrome-extension-ai-agent-rajput) for keys

#### 2. Install Dependencies:
```bash
pip install flask linkedin-api google-generativeai openai python-dotenv selenium
```

---

### **Step 2: Local Flask Server (`server.py`)**
```python
from flask import Flask, request, jsonify
from dotenv import load_dotenv
import os
import openai
import google.generativeai as genai
from linkedin_api import Linkedin  # Unofficial API

load_dotenv()

app = Flask(__name__)
openai.api_key = os.getenv("OPENAI_API_KEY")
genai.configure(api_key=os.getenv("GEMINI_API_KEY"))

# LinkedIn Auth
linkedin = Linkedin(os.getenv("LINKEDIN_EMAIL"), os.getenv("LINKEDIN_PASSWORD"))

def analyze_post(draft):
    # Scrape top posts from your network (example)
    feed = linkedin.get_feed_posts(count=10)
    top_posts = [p["commentary"]["text"] for p in feed if p.get("socialDetail", {}).get("totalSocialActivityCount", 0) > 50]
    
    # GPT-4 Analysis
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": f"Compare this draft to trending LinkedIn posts: {top_posts}"},
            {"role": "user", "content": f"Suggest 3 edits to improve engagement for: {draft}"}
        ]
    )
    return response.choices[0].message.content

@app.route('/optimize', methods=['POST'])
def optimize():
    data = request.json
    draft = data.get("draft")
    suggestions = analyze_post(draft)
    return jsonify({"suggestions": suggestions})

if __name__ == '__main__':
    app.run(port=5000)
```

---

### **Step 3: Chrome Extension**
#### 1. Folder Structure:
```
linkedin-optimizer/
├── manifest.json
├── background.js
├── content.js
└── popup.html
```

#### 2. `manifest.json`:
```json
{
  "manifest_version": 3,
  "name": "LinkedIn Post Optimizer",
  "version": "1.0",
  "permissions": ["activeTab", "scripting"],
  "action": {
    "default_popup": "popup.html"
  },
  "content_scripts": [{
    "matches": ["https://www.linkedin.com/*"],
    "js": ["content.js"]
  }]
}
```

#### 3. `content.js` (Injects Button):
```javascript
// Wait for LinkedIn composer to load
setInterval(() => {
  const composer = document.querySelector('.ql-editor');
  if (composer && !document.getElementById('ai-optimize-btn')) {
    const btn = document.createElement('button');
    btn.id = 'ai-optimize-btn';
    btn.textContent = '✨ Optimize';
    btn.style.marginLeft = '10px';
    btn.onclick = async () => {
      const draft = composer.innerHTML;
      const response = await fetch('http://localhost:5000/optimize', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ draft })
      });
      const data = await response.json();
      alert("AI Suggestions:\n" + data.suggestions);
    };
    composer.parentNode.appendChild(btn);
  }
}, 1000);
```

#### 4. `popup.html` (Optional UI):
```html
<!DOCTYPE html>
<html>
<body>
  <h3>LinkedIn Optimizer</h3>
  <p>Click "✨ Optimize" in LinkedIn's post composer</p>
</body>
</html>
```

---

### **Step 4: Run the System**
1. Start Flask server:
   ```bash
   python server.py
   ```
2. Load extension in Chrome:
   - Go to `chrome://extensions/`
   - Enable Developer Mode → "Load unpacked" → Select folder

---

### **Step 5: Test It**
1. Open LinkedIn → Create new post
2. Click **"✨ Optimize"** button next to the text box
3. Get AI suggestions like:
   > "1. Add a hook question in first line\n2. Include 2-3 hashtags\n3. Add a CTA like 'What do you think?'"

---

### **⚠️ Important Notes**
1. **LinkedIn API Limits**: The unofficial API may get blocked. Alternatives:
   - Use Selenium for scraping (slower but reliable)
   - Official LinkedIn API requires approval
2. **Error Handling**: Add try-catch blocks for API failures
3. **Privacy**: Never store user posts on your server

---

### **🚀 Advanced Features**
1. **Tone Selector** (Professional/Casual):
   ```python
   # Modify analyze_post()
   prompt = f"Suggest edits in {tone} tone: {draft}"
   ```
2. **Hashtag Recommender**:
   ```python
   hashtags = linkedin.get_hashtag_suggestions(draft)
   ```
3. **Engagement Predictor**:
   ```python
   response = openai.ChatCompletion.create(
       model="gpt-4",
       messages=[{"role": "user", "content": f"Rate engagement potential (1-10) of this post: {draft}"}]
   )
   ```
