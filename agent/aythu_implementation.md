# Rythu AI: RAG vs Agentic AI Architecture



### 4. AI Assistant

Recommended approach: RAG plus Agentic AI.

Inputs:

- Farmer question.
- Language: Telugu or English.
- Optional image.
- Farmer profile.
- District/village.
- Crop details.
- Chat history.

Processing:

- Classify user question:
  - Crop knowledge.
  - Disease.
  - Fertilizer.
  - Government scheme.
  - Weather/task planning.
  - Mandi/price.
  - General farming.
- If question needs knowledge, use RAG.
- If question needs live data or decisions, use tools through an agent.
- Generate final answer in farmer-friendly language.

Outputs:

- Answer in Telugu/English.
- Step-by-step advice.
- Safety warning where needed.
- Follow-up question if data is missing.
- Suggested next action.

AI needed:

- RAG for trusted agriculture/scheme answers.
- Agentic AI for personalized decisions and tool calls.

Implementation:

- RAG knowledge bases:
  - Crop guides.
  - Government schemes.
  - Disease treatment.
  - Fertilizer schedules.
  - Pesticide safety.
- Agent tools:
  - `search_knowledge_base`
  - `get_weather`
  - `get_crop_profile`
  - `get_scheme_eligibility`
  - `get_mandi_price`
  - `save_chat_message`
- Use structured answer format:

```json
{
  "answer": "...",
  "confidence": "medium",
  "sources": ["crop_guide_paddy", "weather_forecast"],
  "actions": ["Irrigate today", "Avoid spraying tomorrow"],
  "needsFollowUp": false
}
```

### 5. Weather Alerts

Recommended approach: Weather API plus Agentic AI.

Inputs:

- Latitude.
- Longitude.
- District/location.
- Crop.
- Crop stage.
- Sowing date.
- Forecast data.

Processing:

- Fetch weather from weather API.
- Normal logic extracts temperature, rain chance, humidity, wind, sunrise, sunset.
- Agentic AI decides farming action based on crop and weather:
  - Spray or do not spray.
  - Irrigate or wait.
  - Harvest or delay.
  - Drying safe or unsafe.

Outputs:

- Weather details.
- 5-day forecast.
- Rain chance.
- Humidity.
- Wind.
- Sunrise/sunset.
- Farming tip.
- Alert severity.
- Recommended action.

AI needed:

- No RAG for raw weather.
- Agentic AI useful for crop-specific interpretation.

Implementation:

- Weather service:
  - `GET /api/weather/forecast?lat=&lon=&days=`
- Agent tool:
  - `get_weather_forecast`
  - `get_crop_stage`
- Rules can be hybrid:
  - Rain above 60 percent: avoid spraying/drying.
  - Temperature above 38 C: irrigation warning.
  - Wind above threshold: avoid pesticide spraying.
- Agent explains the rule in simple Telugu/English.

### 6. Farming Guide / Learn

Recommended approach: RAG.

Inputs:

- Selected crop.
- Selected language.
- Farmer question about crop.
- Optional district/season context.

Processing:

- Retrieve crop guide content from knowledge base.
- Pull relevant sections:
  - Soil.
  - Season.
  - Duration.
  - Irrigation.
  - Fertilizer.
  - Diseases.
  - Yield.
  - Tips.
- Summarize based on user query.

Outputs:

- Crop guide.
- Best season.
- Soil type.
- Fertilizer schedule.
- Irrigation schedule.
- Disease/cure info.
- Expected yield.
- Practical tips.

AI needed:

- RAG strongly recommended.
- Agentic AI optional only if combining with user crop dates/weather.

Implementation:

- Store crop guides as structured documents:

```json
{
  "crop": "paddy",
  "section": "fertilizer",
  "content": "..."
}
```

- Embed documents into vector DB.
- Retrieve top relevant chunks.
- Generate answer with source references.
- Keep crop facts editable by agriculture experts.

### 7. Crop Tracker

Recommended approach: Agentic AI plus normal app logic.

Inputs:

- Crop name.
- Sowing date.
- Acres.
- Location.
- Crop stage.
- Weather alert toggle.
- Daily reminder toggle.
- Farmer notes.
- VIP status.

Processing:

- Normal logic stores tracked crop.
- Calculate days since sowing.
- Estimate crop stage.
- Fetch weather.
- Agentic AI creates:
  - Today's task.
  - This week's plan.
  - Risk alerts.
  - Fertilizer/pest reminders.
- RAG can be used to fetch crop-stage guidance.

Outputs:

- Crop progress.
- Current stage.
- Today's task.
- Weather alert.
- Notes.
- AI insights.
- Fertilizer schedule.
- Pest/disease warning.
- Harvest reminder.

AI needed:

- Agentic AI is useful.
- RAG is useful for crop-stage knowledge.

Implementation:

- Data model:

```json
{
  "crop": "paddy",
  "sowingDate": "2026-06-01",
  "acres": 3,
  "stage": "growing",
  "location": "Nalgonda",
  "weatherAlerts": true
}
```

- Agent tools:
  - `get_crop_calendar`
  - `get_weather_forecast`
  - `get_farmer_notes`
  - `save_crop_task`
- Output:

```json
{
  "todayTask": "Check water level and avoid pesticide spray due to rain.",
  "priority": "important",
  "reason": "Rain chance is high tomorrow.",
  "nextReminder": "2026-06-30"
}
```

### 8. Government Schemes

Recommended approach: RAG plus Agentic AI.

Inputs:

- Land ownership.
- Land size.
- Farmer type.
- Age.
- Income.
- Occupation.
- Category.
- Gender.
- Documents available.
- District/state.

Processing:

- RAG retrieves scheme rules, documents, benefits, official links, and application steps.
- Agentic AI compares farmer inputs against scheme rules.
- It separates eligible and not eligible schemes.
- It explains why the farmer qualifies or does not qualify.
- It tells what documents/actions are missing.

Outputs:

- Eligible schemes.
- Not eligible schemes.
- Estimated benefits.
- Eligibility reason.
- Missing requirements.
- Required documents.
- How to apply.
- Official website.
- Helpline.

AI needed:

- RAG for scheme knowledge.
- Agentic AI for eligibility reasoning.

Implementation:

- Store each scheme as structured data:

```json
{
  "scheme": "PM-KISAN",
  "benefit": "Rs 6,000/year",
  "eligibilityRules": ["land_owner", "not_income_tax_payer"],
  "documents": ["Aadhaar", "Bank Account", "Land Records"],
  "applySteps": ["Visit portal", "Complete eKYC"]
}
```

- Use deterministic rules where possible.
- Use agent to explain results, not to invent rules.
- Output should include source IDs for every scheme.

### 9. Paddy Quality Scanner

Recommended approach: Vision AI plus Agentic AI plus RAG.

Inputs:

- Paddy photo.
- Paddy variety.
- Lighting/photo quality.
- Farmer district.
- Optional mandi context.
- VIP status.

Processing:

- Vision model analyzes image:
  - Damaged grains.
  - Foreign matter.
  - Grain uniformity.
  - Moisture clues.
  - Visual condition.
- Agentic AI converts image results into:
  - Grade.
  - Risk level.
  - Price impact.
  - Recommendations.
- RAG can explain grading standards and quality improvement tips.

Outputs:

- Grade A/B/C/D.
- Quality score.
- Damaged grain estimate.
- Foreign matter estimate.
- Moisture warning.
- Milling recovery estimate.
- Estimated price range.
- Confidence.
- Retake tips.
- Selling advice.

AI needed:

- Vision AI required.
- Agentic AI useful for interpretation and next action.
- RAG useful for grading explanation and improvement tips.

Implementation:

- Backend endpoint: `POST /scan/paddy`.
- Validate image size and quality.
- Run vision model.
- Return structured result:

```json
{
  "grade": "B",
  "confidence": 0.78,
  "damagedGrains": "medium",
  "foreignMatter": "low",
  "moistureRisk": "medium",
  "millingRecovery": "62-66%",
  "recommendations": ["Dry for one more day", "Remove visible foreign matter"]
}
```

### 10. Crop Disease Scanner

Recommended approach: Vision AI plus RAG plus Agentic AI.

Inputs:

- Crop photo.
- Crop selected by farmer, if any.
- Crop automatically detected by AI.
- Symptoms visible in image.
- Acres/field size.
- Crop stage.
- Weather.
- VIP status.

Processing:

- Vision model detects crop and disease.
- RAG retrieves disease facts, symptoms, causes, and treatments.
- Agentic AI creates a treatment plan using:
  - Disease.
  - Severity.
  - Crop stage.
  - Acres.
  - Weather conditions.
  - Organic/chemical preference if available.
- VIP users get exact dosage and full treatment plan.

Outputs:

- Crop detected.
- Disease detected.
- Confidence.
- Severity.
- Symptoms observed.
- Cause.
- Why it happened.
- Organic treatment.
- Chemical treatment.
- Dosage.
- Spray timing.
- Cost estimate.
- Expert confirmation warning.

AI needed:

- Vision AI required.
- RAG strongly recommended.
- Agentic AI required for dosage/action planning.

Implementation:

- Backend endpoint: `POST /scan/disease`.
- Vision result:

```json
{
  "crop": "chilli",
  "disease": "thrips",
  "confidence": 0.84,
  "severity": "medium"
}
```

- Agent then calls:
  - `retrieve_disease_treatment`
  - `get_weather_forecast`
  - `calculate_dosage`
- Final output should include warnings and source references.

### 11. Mandi Prices

Recommended approach: API/database plus Agentic AI, optional RAG.

Inputs:

- District.
- Mandi/market.
- Crop/commodity.
- Date.
- Quality grade.
- Quantity.
- Paddy scanner result, if available.

Processing:

- Live prices should come from a trusted API or database, not from a language model.
- Agentic AI can compare:
  - Today's price.
  - Nearby mandi prices.
  - Quality grade.
  - Farmer quantity.
  - Weather/storage urgency.
- RAG can explain mandi rules, grading terms, and selling tips.

Outputs:

- Today's mandi price.
- Min/max/modal price.
- Best nearby mandi.
- Sell/wait suggestion.
- Price explanation.
- Quality-based negotiation tip.

AI needed:

- No RAG for live price values.
- RAG optional for explaining market rules.
- Agentic AI useful for sell/wait advice.

Implementation:

- Data source options:
  - Government mandi API.
  - Scraped official market data.
  - Admin-updated price database.
- Endpoint:
  - `GET /mandi/prices?district=&crop=&date=`
- Agent output:

```json
{
  "recommendation": "Sell today if grade is A/B; prices are strong.",
  "bestMarket": "Nalgonda",
  "priceRange": "Rs 2200-2400/quintal",
  "reason": "Current price is above weekly average."
}
```

### 12. Voice AI Call

Recommended approach: Agentic AI plus speech-to-text and text-to-speech.

Inputs:

- Microphone audio.
- Language.
- User profile.
- Crop data.
- Conversation history.
- VIP/free-trial status.

Processing:

- Convert speech to text.
- Send text to the same AI assistant agent.
- Agent calls tools or RAG as needed.
- Convert answer back to speech.
- Enforce call duration limits.

Outputs:

- Spoken farming advice.
- Telugu/English conversation.
- Follow-up questions.
- Tool-based answers for weather, crop, schemes, etc.
- Trial limit or VIP limit notice.

AI needed:

- Agentic AI required.
- RAG used when questions need knowledge.
- Speech models required for voice interface.

Implementation:

- Pipeline:
  - Speech-to-text.
  - Agent/RAG reasoning.
  - Text-to-speech.
  - Audio streaming.
- Reuse the AI Assistant backend so voice and chat give consistent answers.

### 13. Profile And Account

Recommended approach: Normal app logic.

Inputs:

- Full name.
- Phone number.
- Village/Mandal.
- District.
- Email.
- Profile photo.

Processing:

- Save profile to database.
- Use profile fields to personalize other features.
- Use district/village for schemes and AI context.
- Use account security rules for delete account.

Outputs:

- Saved profile.
- Personalized dashboard.
- Auto-filled scheme fields.
- AI context.
- Account deleted state.

AI needed:

- No RAG needed.
- No agentic AI needed.

Implementation:

- CRUD APIs.
- Firebase/Firestore profile document.
- Storage bucket for profile photo.
- Security rules so users can access only their own profile.

### 14. VIP And Payments

Recommended approach: Normal app logic.

Inputs:

- Selected plan.
- User ID.
- Razorpay payment response.
- Payment ID.
- Signature.

Processing:

- Start Razorpay checkout.
- Verify payment signature on backend.
- Update subscription status.
- Store payment ID and expiry date.
- Unlock premium limits.

Outputs:

- VIP active status.
- Expiry date.
- Renewal warning.
- Payment success/failure.
- Premium feature access.

AI needed:

- No RAG needed.
- No agentic AI needed for payment itself.
- AI can explain payment failure, but must not decide payment validity.

Implementation:

- Backend endpoints:
  - `POST /payments/create-order`
  - `POST /payments/verify`
  - `GET /subscription/status`
- Verify all payments server-side.
- Never trust frontend-only payment success.

### 15. Privacy And Data Handling

Recommended approach: Normal app logic plus governance.

Inputs:

- Profile data.
- Chat history.
- Crop data.
- Scan history.
- GPS location.
- Payment ID.
- Delete account request.
- Clear chat request.

Processing:

- Store only necessary data.
- Restrict access by user ID.
- Delete/clear data when user requests.
- Send AI requests only with needed context.
- Log consent where required.

Outputs:

- Clear chat history.
- Delete account.
- Privacy policy.
- Data access control.

AI needed:

- No RAG needed.
- No agentic AI needed.

Implementation:

- Firestore security rules.
- Data retention policy.
- Delete account job.
- User-facing privacy controls.
- Audit logs for important account actions.

## Recommended Overall Architecture

### Knowledge Base For RAG

Create separate collections:

- `crop_guides`
- `scheme_rules`
- `disease_treatments`
- `fertilizer_schedules`
- `pesticide_safety`
- `mandi_education`

Each document should contain:

- Title.
- Language.
- State/district relevance.
- Crop/scheme name.
- Content.
- Source URL or source name.
- Last updated date.
- Expert verification status.

### Agent Tools

Useful tools:

- `get_weather_forecast(location, days)`
- `get_user_profile(user_id)`
- `get_tracked_crops(user_id)`
- `get_crop_stage(crop, sowing_date)`
- `search_knowledge_base(query, filters)`
- `check_scheme_eligibility(farmer_profile)`
- `get_mandi_prices(district, crop)`
- `calculate_dosage(crop, disease, acres)`
- `save_chat_message(user_id, message)`
- `create_crop_reminder(user_id, crop_id, task)`

### Safety Rules

- Do not invent government eligibility rules.
- Always show official source links for scheme answers.
- For pesticide/fungicide dosage, show warnings and recommend local agriculture officer confirmation.
- For payments, never use AI to verify payment success.
- For medical/safety-like agricultural chemical advice, prefer conservative guidance.
- For live prices and weather, use APIs/databases, not model memory.

## Final Recommendation

Use this split:

- RAG for trusted knowledge.
- Agentic AI for personalized decisions and tool use.
- Vision AI for scanners.
- Normal backend logic for auth, payments, profile, privacy, and static navigation.

The strongest Rythu AI architecture is not "only RAG" or "only agents." It should combine both:

- RAG gives correct knowledge.
- Agentic AI uses that knowledge with farmer-specific data.
- APIs provide live facts.
- Normal app logic handles security-critical operations.

