# -Wedding-Card-Shop-Dynamic-Pricing-Automation_using_N8N
Price calculation me ye rules apply ho rhy hen:  per card base_price Google Sheet se aayega har card ki apni min_qty ho agar order quantity > 50 ho to per card Rs 5 ka discount agar quantity min_qty se kam ho to user ko warning / error.

Project ka naam
Wedding Card Shop – Dynamic Pricing Automation

2. Business Problem
Shaadi cards ki multiple varieties hain:

har card ka alagh card_code hai
base_price (per card) different hai
minimum quantity bhi card ke hisaab se different hai
kuch cards inactive bhi ho sakte hain
Shop owner chahte hain:

User website pe card code + quantity daale aur usi waqt correct price mil jaye.

Price calculation me ye rules apply hon:

per card base_price Google Sheet se aayega
har card ki apni min_qty ho
agar order quantity > 50 ho to per card Rs 5 ka discount
agar quantity min_qty se kam ho to user ko warning / error.
System itna flexible ho ke:

Card catalog easily sheet me update ho jaye (code change na karna pade).
Discount rules change karna ho to sirf config values change hon.
3. System Architecture
3.1 Components
Front‑end Web Form

Simple HTML/CSS/JS page
Inputs:
Card Code
Quantity
Button: Get Price
Response show karti hai:
Card Code
Quantity
Price per Card
Total Price
(optional) Discount per Card
Backend Automation – n8n Workflow

Workflow ka naam: Wedding Card Shop Pricing Automation
n8n HTTP endpoint ko front‑end se call kiya jata hai.
Logic, validations, discounts sab yahin handle hote hain.
Data Store – Google Sheets

Sheet Card_Prices jisme sari card varieties ka data:

column	description
card_code	unique code (1001, 1002, C101, HQ20, …)
quantity	default / pack quantity (informational)
base_price	per card base price (50, 60, 300, …)
min_qty	is card ki minimum order quantity
status	active / inactive
4. Data Flow (End‑to‑End)
User website form pe card_code + quantity enter karta hai → Get Price.
Front‑end ye data JSON ki form me n8n Webhook ko POST karta hai.
n8n workflow request receive karta hai, sheet se relevant card dhoondta hai, business rules apply karke price calculate karta hai.
n8n response JSON bhejta hai:
JSON

{
  "card_code": "1002",
  "quantity": 60,
  "base_price": 60,
  "min_qty": 30,
  "price_per_card": 55,
  "discount_per_card": 5,
  "total_price": 3300,
  "applied_discount": true
}
Front‑end is response ko read karke result UI pe show kar deta hai.
5. n8n Workflow – Nodes Detail
5.1 Receive Card Order (Webhook)
Method: POST
Expected body:
JSON

{
  "card_code": "1002",
  "quantity": 60
}
5.2 Workflow Configuration (Set / Function node)
Yahan aap constants rakh sakte ho, jaise:
discount rules
messages, etc.
Saath hi, user wali quantity ko normalize karke aage pass kar sakte ho
(aap ke case me code node directly Workflow Configuration se quantity read kar raha hai).
5.3 Lookup Card In Card_Prices (Google Sheets node)
Action: Read / Lookup Row
Condition: card_code == {{ card_code jo webhook se aaya }}
Output: us card ki poori row:
card_code
base_price
min_qty
status
etc.
5.4 Check Card Exists (IF node)
Condition:
Agar lookup node se row na mile ya status != active ho → “Error – Invalid Card Code” branch.
Warna aage Continue.
5.5 Check Minimum Quantity (IF node)
Left: user quantity (Webhook/Workflow Config se)
Right: min_qty (Sheet row se)
Operator: >=
false → “Error – Below Minimum Quantity” node
true → Calculate Price with Discounts node
5.6 Calculate Price with Discounts (Code node)
Ye main calculation node hai.
Isme aapka optimized final code chal raha hai:

Logic Summary
Inputs:

card_code, base_price, min_qty → Google Sheet se
quantity → user order quantity (Workflow Configuration / webhook se)
Validation:

card_code missing ho to error message.
Agar quantity <= 0 ho to total 0 with message.
Agar quantity < min_qty ho to warning + base price ke sath hi result.
Discount Rule:

Global config:
JavaScript

BULK_DISCOUNT_MIN_QTY = 50;
BULK_DISCOUNT_PER_CARD = 5;
Agar quantity > 50:
JavaScript

pricePerCard = base_price - 5;
discount_per_card = 5;
applied_discount = true;
Agar quantity <= 50:
JavaScript

pricePerCard = base_price;
discount_per_card = 0;
applied_discount = false;
Final Calculation:

JavaScript

totalPrice = quantity * pricePerCard;
Output fields:

JSON

{
  "card_code": "...",
  "quantity": 60,
  "base_price": 60,
  "min_qty": 30,
  "price_per_card": 55,
  "total_price": 3300,
  "discount_per_card": 5,
  "applied_discount": true
}
5.7 Format Success Response
Agar aap extra formatting chahte ho (property names change karna, message add karna), ye node karta hai.
Lekin aap simple case me Code node ka output direct Webhook ko bhi de sakte ho.
5.8 Return Price Response (Webhook Response)
n8n ka Response node jo front‑end ko JSON wapis bhejta hai.
Status code: 200 (success) ya 4xx for error (optional).
6. Front‑End Web Form (high‑level)
Form fields:

HTML

<input type="text" id="cardCode" placeholder="Card Code">
<input type="number" id="quantity" placeholder="Quantity">
<button id="btnGetPrice">Get Price</button>
JavaScript (simplified):

JavaScript

document.getElementById('btnGetPrice').addEventListener('click', async () => {
  const cardCode = document.getElementById('cardCode').value;
  const quantity = document.getElementById('quantity').value;

  const res = await fetch('N8N_WEBHOOK_URL', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ card_code: cardCode, quantity }),
  });

  const data = await res.json();

  // show result
  document.getElementById('result').innerHTML = `
    Card Code: ${data.card_code}<br>
    Quantity: ${data.quantity}<br>
    Price per Card: Rs ${data.price_per_card}<br>
    Total Price: Rs ${data.total_price}<br>
    ${data.discount_per_card && data.discount_per_card > 0
      ? `Discount per Card: Rs ${data.discount_per_card}`
      : ''
    }
  `;
});
7. Key Features (jo aap CV / portfolio me likh sakte ho)
Google Sheets ko lightweight database ki tarah use karke dynamic pricing.
Low‑code / no‑code backend using n8n for:
data validation,
business rules (min quantity, bulk discount),
error handling.
Config‑driven discount rule:
Bulk discount easily change ho sakta hai:
BULK_DISCOUNT_MIN_QTY aur BULK_DISCOUNT_PER_CARD values adjust karke.
Front‑end se real‑time price calculation:
user card code change kare ya quantity badlaye, system turant new price return karta hai.
8. Possible Future Enhancements
