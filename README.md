# Xero-Financial-Analyst
This Python Flask app integrates with the Xero Accounting API to fetch real-time financial data (Profit &amp; Loss, Balance Sheet), and uses OpenAI GPT-4o-mini to generate automated insights and analysis.

✨ Features
🔐 Secure OAuth2 login with Xero

📈 Fetches Profit and Loss and Balance Sheet reports

🤖 Analyzes reports using OpenAI GPT-4o-mini

🌐 Displays results in your web browser

🧠 AI-generated summary, trends, risks, and strategic suggestions

🚀 Live Demo (Local)
bash
clone Repo
cd xero-gpt-financial-analyzer
pip install -r requirements.txt
python app.py
Then open: http://localhost:5000

⚙️ Setup
1. Clone the Repo
bash
Copy
Edit
git clone https://github.com/your-username/xero-gpt-financial-analyzer.git
2. Install Dependencies
bash
Copy
Edit
pip install flask requests requests_oauthlib openai
3. Set Your Keys
Edit app.py:

python
Copy
Edit
CLIENT_ID = "your_xero_client_id"
CLIENT_SECRET = "your_xero_client_secret"
OPENAI_API_KEY = "sk-...your_openai_key..."
Make sure your redirect URI matches:

bash
Copy
Edit
http://localhost:5000/callback
🧠 Powered By
Xero Developer API

OpenAI GPT-4o-mini

Flask

📌 To-Do (Future Ideas)
 Email reports to team

 Export analysis to PDF or Excel

 Schedule weekly analysis

 Add login and role-based access


 Here the detailed code: 
 import os
import json
import requests
import webbrowser
from flask import Flask, redirect, request
from requests_oauthlib import OAuth2Session
from openai import OpenAI

# ✅ Allow insecure HTTP for local testing
os.environ['OAUTHLIB_INSECURE_TRANSPORT'] = '1'

# === CONFIGURATION ===
CLIENT_ID = 'Your_CLIENT_ID'
CLIENT_SECRET = 'Your_CLIENT_SECRET'
REDIRECT_URI = 'http://localhost:5000/callback'
OPENAI_API_KEY = 'OPENAI_API_KEY'

AUTH_BASE_URL = 'https://login.xero.com/identity/connect/authorize'
TOKEN_URL = 'https://identity.xero.com/connect/token'
API_BASE_URL = 'https://api.xero.com/api.xro/2.0'

app = Flask(__name__)

@app.route('/')
def login():
    xero = OAuth2Session(CLIENT_ID, redirect_uri=REDIRECT_URI, scope=['accounting.reports.read'])
    auth_url, _ = xero.authorization_url(AUTH_BASE_URL)
    return redirect(auth_url)

@app.route('/callback')
def callback():
    try:
        xero = OAuth2Session(CLIENT_ID, redirect_uri=REDIRECT_URI)
        token = xero.fetch_token(
            TOKEN_URL,
            client_secret=CLIENT_SECRET,
            authorization_response=request.url,
        )

        access_token = token['access_token']
        headers = {
            "Authorization": f"Bearer {access_token}",
            "Accept": "application/json"
        }

        # Get Tenant ID
        conn_resp = requests.get("https://api.xero.com/connections", headers=headers).json()
        if not conn_resp or not conn_resp[0].get("tenantId"):
            return "❌ Error: Unable to get tenant ID from Xero"

        tenant_id = conn_resp[0]["tenantId"]
        headers["Xero-tenant-id"] = tenant_id

        # === Get Financial Reports ===
        pl_data = requests.get(f"{API_BASE_URL}/Reports/ProfitAndLoss", headers=headers).json()
        bs_data = requests.get(f"{API_BASE_URL}/Reports/BalanceSheet", headers=headers).json()

        # === Use OpenAI GPT-4 (new SDK) ===
        client = OpenAI(api_key=OPENAI_API_KEY)
        prompt = f"""
        Could you look over the following Xero financial reports?

        Profit and Loss:
        {json.dumps(pl_data)}

        Balance Sheet:
        {json.dumps(bs_data)}

        Please provide:
        - Summary
        - Key trends
        - Anomalies or risks
        - Strategic suggestions
        """

        chat_completion = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a financial analyst."},
                {"role": "user", "content": prompt}
            ]
        )

        result = chat_completion.choices[0].message.content
        return f"<h2>📊 Financial Insights from GPT-4</h2><pre>{result}</pre>"

    except Exception as e:
        import traceback
        tb = traceback.format_exc()
        print("❌ ERROR in callback:\n", tb)
        return f"<h2>❌ Internal Server Error</h2><pre>{tb}</pre>"

if __name__ == '__main__':
    app.debug = True
    print("✅ Flask app running at http://localhost:5000")
    webbrowser.open("http://localhost:5000")
    app.run(port=5000)
