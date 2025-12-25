import os
from fastapi import FastAPI, Request, Header, HTTPException
from openai import OpenAI
from fpdf import FPDF
from supabase import create_client
import stripe

# Initialize APIs from Environment Variables
app = FastAPI()
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
supabase = create_client(os.getenv("SUPABASE_URL"), os.getenv("SUPABASE_KEY"))
stripe.api_key = os.getenv("STRIPE_SECRET_KEY")

@app.get("/")
def home():
    return {"status": "Nexus Factory Online"}

# --- ASSET GENERATION ---
@app.post("/generate-asset")
async def generate_asset(topic: str):
    # 1. AI Content Creation
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": f"Write a detailed 2000-word guide on {topic}"}]
    )
    content = response.choices[0].message.content

    # 2. PDF Creation
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", size=12)
    pdf.multi_cell(0, 10, content)
    file_name = f"{topic.replace(' ', '_')}.pdf"
    pdf.output(file_name)

    # 3. Upload to Supabase Storage
    with open(file_name, 'rb') as f:
        supabase.storage.from_("assets").upload(file_name, f)
    
    # 4. Save to Database
    asset_url = supabase.storage.from_("assets").get_public_url(file_name)
    supabase.table("digital_assets").insert({"name": topic, "url": asset_url}).execute()
    
    return {"message": "Asset Created", "url": asset_url}

# --- INCOME RETRIEVAL (STRIPE WEBHOOK) ---
@app.post("/webhook")
async def stripe_webhook(request: Request, stripe_signature: str = Header(None)):
    payload = await request.body()
    try:
        event = stripe.Webhook.construct_event(payload, stripe_signature, os.getenv("WEBHOOK_SECRET"))
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

    if event['type'] == 'checkout.session.completed':
        session = event['data']['object']
        # Log income to Supabase
        supabase.table("sales_logs").insert({
            "amount": session['amount_total'] / 100,
            "source": "Stripe Sale"
        }).execute()
    
    return {"status": "success"}


import os
import stripe
from supabase import create_client

stripe.api_key = os.getenv("STRIPE_SECRET_KEY")
supabase = create_client(os.getenv("SUPABASE_URL"), os.getenv("SUPABASE_KEY"))

def sync_vault():
    # Retrieve balance from Stripe
    balance = stripe.Balance.retrieve()
    available = balance['available'][0]['amount'] / 100
    
    # Update Supabase Vault
    supabase.table("vault_stats").upsert({
        "id": 1,
        "retrievable_cash": available,
        "last_sync": "now()"
    }).execute()
    print(f"Vault Synced: ${available} available for retrieval.")

if __name__ == "__main__":
    sync_vault()

DEPENDENCIES
fastapi
uvicorn
openai
fpdf
supabase
stripe
python-multipart

