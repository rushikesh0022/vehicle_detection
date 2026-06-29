# Rythu AI Website Functionality Audit

Audit date: 2026-06-29  
Website checked: https://www.rythuai.in/home

## Scope And Access Notes

- The public homepage, login page, privacy page, and shipped frontend bundle were inspected.
- The supplied account credentials were valid when checked against Firebase Authentication with the site referer, but the visible website login form repeatedly returned `Network error. Check your internet connection.` in the browser session. Because of that, dashboard-only pages could not be fully operated as a logged-in user.
- Auth-gated routes such as `/home` and `/voice-test` redirected to the landing/login flow.
- The details below combine directly visible public pages with feature labels, route behavior, and data structures shipped in the site's JavaScript bundle.

## Public Website And Onboarding

### Homepage

The homepage presents Rythu AI as a Telangana farming assistant with Telugu and English support.

Visible sections:

- Hero: "Your Personal AI Expert For Your Farm"
- Language switch: Telugu / English
- Cookie banner: Accept / Decline
- Navigation: Features, Pricing, Login
- Feature cards
- "How Does It Work?" onboarding steps
- Plans and pricing
- Farmer testimonials
- Mission, vision, founder section
- Footer with quick links and contact email

### Onboarding Flow

The homepage describes a three-step workflow:

1. Create an account with email.
2. Register crop name and sowing date.
3. Get daily tasks, weather alerts, and AI chat help.

### Authentication

Visible and bundled authentication features:

- Login by Email.
- Login by Phone tab is visible, although OTP details were not fully exposed publicly.
- Signup flow with name, email address, and password.
- Forgot password flow.
- Password reset screen with new password and confirm password fields.
- Email verification screen:
  - Verify email title.
  - Instructions to click the email link and return.
  - Resend verification link.
  - "I've verified, enter app" action.
  - Login with different account action.

Authentication error messages include:

- Email already registered.
- Invalid email.
- No account found.
- Invalid email or password.
- Weak password.
- Wrong password.
- User disabled.
- Network error.

## Main App Navigation

The authenticated app navigation labels exposed in the bundle are:

- Home
- Learn
- Schemes
- Weather
- Track Crop
- Scan

The dashboard/home area also exposes:

- Welcome message by time of day.
- User name and farmer tag.
- Primary crop and district display.
- Today's message / quote.
- VIP expiry warning if the VIP pass is close to expiry.
- Quick access cards.
- Upgrade / renew prompts.

## AI Assistant

The AI assistant is branded as "Rythu AI Expert".

Core functions:

- Ask farming questions in Telugu or English.
- Text chat input.
- Suggested questions:
  - "What should I plant in this season?"
  - "My crops have yellow spots on leaves"
  - "Which government scheme is best for me?"
  - "Tips for organic farming"
- Optional image attachment for diagnosis.
- Chat history context is sent with recent messages.
- Farmer context is sent with requests:
  - User ID
  - Name
  - District
  - Village
  - Language
- AI replies are saved to chat history.
- Clear chat history.
- Delete chat.
- Retry last user message.
- Sound playback toggle for AI replies.
- Server timeout handling.
- Network error message if chat fails.

Voice input inside chat:

- Uses browser speech recognition when supported.
- Language switches between `te-IN` and `en-IN`.
- Listening placeholder appears while recording.
- Captured speech is placed into the chat input for review before sending.
- Browser unsupported error is shown if speech recognition is unavailable.

Backend endpoint visible in the bundle:

- `POST /api/ai/chat`

## Weather Alerts

Weather is one of the most detailed feature screens in the app.

Data source behavior visible in the bundle:

- Calls `/api/weather/forecast?lat=<latitude>&lon=<longitude>&days=5`.
- Uses saved or detected location latitude and longitude.
- Dashboard also calls the weather endpoint with `days=3` for short summary cards.

Weather details shown:

- Location name.
- Today's weather / selected day's weather.
- Weather condition text.
- Weather icon based on weather code.
- Maximum temperature.
- Minimum temperature.
- Rain chance as a percentage.
- Humidity as a percentage.
- Wind speed in km/h.
- Feels-like temperature.
- Sunrise time.
- Sunset time.
- 5-day forecast list.
- Date for each forecast day.
- Rain chance for each forecast day.
- Max/min temperature for each forecast day.
- Retry button if loading fails.

Weather condition labels include:

- Clear Sky
- Mainly Clear
- Partly Cloudy
- Overcast
- Light Drizzle
- Slight Rain
- Moderate Rain
- Heavy Rain
- Rain Showers
- Thunderstorm

Farm-specific weather tips:

- If rain chance is above 60 percent: delay harvesting or drying activities.
- If max temperature is above 38 C: ensure crops are adequately irrigated.
- Otherwise: conditions are suitable for farm activities.

Crop tracker weather alerts:

- Weather alerts can be enabled per tracked crop.
- Heavy rain warning: delay fertilizer/pesticide.
- Extreme heat warning: ensure irrigation.

## Farming Guide / Learn

The UI copy says "Complete guide for 13 crops"; however, the shipped crop-guide data contains 22 crop guides.

Guide categories:

- Vegetables:
  - Ladies Finger (Bhendi)
  - Tomato
  - Chilli
  - Onion
  - Brinjal (Eggplant)
  - Bitter Gourd
  - Ginger
  - Garlic
- Leafy Vegetables:
  - Spinach (Palak)
  - Amaranth (Thotakura)
  - Coriander (Kothimira)
- Kharif Crops:
  - Paddy (Rice)
  - Cotton
  - Maize (Corn)
  - Groundnut
  - Turmeric
  - Red Gram (Pigeon Pea/Kandi)
- Rabi Crops:
  - Wheat
  - Bengal Gram (Chickpea)
  - Sunflower
  - Green Gram (Moong)
  - Black Gram (Urad)

Each crop guide can include:

- Soil type.
- Season.
- Duration / crop days.
- Irrigation guidance.
- Fertilizer guidance.
- Expected yield.
- Diseases and cure.
- Pro tips.

## Track Crop

The crop tracker helps farmers manage crops from sowing to harvest.

Supported tracker crop options exposed:

- Paddy
- Tomato
- Chilli
- Cotton
- Onion
- Maize

Add crop fields:

- Crop name.
- Sowing date.
- Acres.
- Location.
- GPS detected label.
- Weather alerts toggle.
- Daily email reminders toggle.

Tracking limits:

- Free users can track up to 2 crops.
- VIP users can track up to 5 crops.

Crop stages:

- Planning
- Sowing
- Growing
- Flowering
- Harvesting
- Harvested

Crop actions:

- Tap a stage to update status.
- Add note.
- View details.
- Save note.
- Remove crop.
- Mark as harvested.

Notes:

- Notes area.
- "No notes yet" state.
- Note types exposed:
  - Observation
  - Issue
  - Action
- Note priorities:
  - Urgent
  - Important
  - Info

Today task and insights:

- Today's task card.
- Day number.
- Total days.
- AI crop insights.
- Cached insight label.
- Action suggested today.
- Plan for this week.
- Read more / read less.
- Premium unlock for more AI insights.

VIP-only tracker insights include:

- AI fertilizer schedules.
- Pest and disease alerts.
- Fertilizer recommendations.
- Pest warnings tailored to the field.

Backend endpoint visible in the bundle:

- `GET /crop/today-task?uid=<user_id>`

## Government Schemes

The public homepage says the app helps with PM-KISAN, Rythu Bandhu/Bharosa, and Rythu Bima. The app copy says "10 schemes - claim free benefits."

### How The Government Scheme Feature Works

The Government Schemes feature works like an eligibility checker. It collects basic farmer, land, income, occupation, and document information, compares those details with each scheme's eligibility rules, and then returns a list of schemes the farmer may qualify for.

General workflow:

1. The farmer opens the Schemes section.
2. The app shows government scheme cards and a "Check Eligibility" flow.
3. Some profile details can be auto-filled, such as name, district, village, and linked email if the profile already has them.
4. The farmer enters or confirms personal and farm details.
5. The farmer marks which important documents are available.
6. The app checks the answers against scheme rules.
7. The app shows eligible schemes, not-eligible schemes, estimated benefit, reasons, required documents, and application steps.

### Inputs The Scheme Checker Takes

Profile and location inputs:

- Farmer name.
- District.
- Village / Mandal.
- Linked email or account identity.

Land and farming inputs:

- Land ownership type:
  - Own land.
  - Tenant farmer.
  - Both owner and tenant.
- Farmer type based on land size:
  - Marginal farmer: less than 2.5 acres.
  - Small farmer: 2.5 to 5 acres.
  - Large farmer: more than 5 acres.
- Land records / Pattadar Passbook availability.
- Crop insurance availability.
- Kisan Credit Card availability.

Personal inputs:

- Category:
  - General
  - SC
  - ST
  - BC
  - Minority
- Gender:
  - Male
  - Female
  - Other
- Age.
- Annual income tier:
  - Less than Rs 1 lakh.
  - Rs 1 lakh to Rs 2.5 lakhs.
  - Rs 2.5 lakhs to Rs 5 lakhs.
  - Above Rs 5 lakhs.
- Occupation:
  - Pure farmer.
  - Private employee.
  - Government job.
  - Pensioner.
  - Professional such as doctor or lawyer.

Document checklist inputs:

- Aadhaar Card.
- Bank Account.
- Kisan Credit Card (KCC).
- Crop Insurance.
- Land Records / Pattadar Passbook.

### Logic Used For Eligibility

The app appears to make rule-based checks using the above inputs. For example:

- Land-owning farmer status matters for PM-KISAN-type benefits.
- Tenant farmer status matters for schemes that support verified tenant farmers.
- Land size affects whether the farmer is marginal, small, or large.
- Age matters for pension or insurance-related schemes.
- Income tier affects eligibility because higher-income groups may be excluded.
- Government job and professional occupation can exclude the farmer from some benefits.
- Income tax payer warning is shown because tax payers are excluded from schemes like PM-KISAN.
- Document availability affects whether the farmer can apply immediately or must collect documents first.

### Outputs The Scheme Checker Gives

The result screen can show:

- Number of eligible schemes.
- Eligible schemes list.
- Not eligible schemes list.
- Estimated total benefit amount.
- Scheme-wise benefit amount.
- Why the farmer is eligible.
- Reason for eligibility or non-eligibility.
- Documents needed.
- How to apply.
- What to do to qualify if currently not eligible.
- Helpline.
- Official website.
- Apply online link.

Example output style:

- "You are eligible for 3 schemes."
- "Estimated Total Benefit: Rs X."
- "Eligible Schemes: PM-KISAN, Rythu Bharosa, Rythu Bima."
- "Reason: Land owner, valid land records, bank account available."
- "Documents Needed: Aadhaar Card, Pattadar Passbook, Bank Account."
- "How to Apply: Visit official portal or contact Mandal Agriculture Office."

Some matching schemes may be hidden behind VIP with a prompt to upgrade to see all matching schemes.

Scheme screen fields:

- Scheme count.
- Benefit.
- Eligibility.
- Required documents.
- How to apply.
- Helpline.
- Website.
- Apply online link.
- General disclaimer that eligibility and amounts may change.
- "All info sourced from official government portals" label.

Detailed scheme cards found in the shipped data:

- PM-KISAN:
  - Yearly income support for land-owning farmers.
  - Amount: Rs 6,000/year.
- PMFBY (Crop Insurance):
  - Crop insurance against calamities.
  - Amount shown as 2 percent premium.
- Kisan Credit Card:
  - Credit for farming and related needs.
  - Interest shown as 4 percent.
- PM Kisan Maan Dhan:
  - Social security for small farmers after 60.
  - Amount: Rs 3,000/month.
- Rythu Bharosa:
  - Investment support.
  - Amount: Rs 12,000/acre.
- Rythu Bima:
  - Life insurance.
  - Amount: Rs 5,00,000.

Additional scheme names recognized by the eligibility mapping:

- Free Electricity / Telangana Free Electricity.
- Crop Loan Waiver / Telangana Crop Loan Waiver.
- PMKSY.
- Soil Health Card Scheme.
- SMAM / Telangana SMAM.
- e-NAM.
- NFSM.
- Subsidized Seeds / Telangana Seed Subsidy.
- PM-KUSUM.

Eligibility checker fields:

- Auto-filled details.
- Personal details.
- Documents checklist.
- Aadhaar Card.
- Bank Account.
- Kisan Credit Card (KCC).
- Crop Insurance.
- Land Records / Pattadar Passbook.
- Own Land.
- Tenant Farmer.
- Land ownership:
  - Land Owner
  - Tenant Farmer
  - Owner and Tenant
- Category:
  - General
  - SC
  - ST
  - BC
  - Minority
- Gender:
  - Male
  - Female
  - Other
- Age.
- Annual income.
- Farmer type:
  - Marginal, less than 2.5 acres.
  - Small, 2.5 to 5 acres.
  - Large, above 5 acres.
- Occupation:
  - Pure Farmer
  - Private Employee
  - Government Job
  - Pensioner
  - Professional, such as doctor or lawyer.
- Annual income tier:
  - Less than Rs 1 lakh.
  - Rs 1 lakh to Rs 2.5 lakhs.
  - Rs 2.5 lakhs to Rs 5 lakhs.
  - Above Rs 5 lakhs.

Eligibility results can show:

- Eligible schemes.
- Not eligible schemes.
- Estimated total benefit.
- Why eligible.
- Reason.
- How to apply.
- Documents needed.
- What to do to qualify.
- More schemes locked behind VIP.

Warnings:

- Government employees except Class IV are generally excluded from most schemes.
- Income tax payers are excluded from schemes like PM-KISAN.

## Scan Lab

The scan area exposes two major scan types.

### Paddy Quality Scanner

Public homepage claim:

- Lab-grade quality check in 10 seconds.

In-app scan copy:

- Take a paddy photo.
- Get Grade A/B/C/D quality rating.
- Get price estimate.
- Get milling recovery estimate.
- AI-powered scan results in 15-30 seconds.

Inputs:

- Paddy variety.
- Paddy photo.
- Capture or upload a paddy photo.
- Photo-ready state.
- Photo size must be less than 1 MB.

Analysis outputs:

- Paddy analysis title.
- AI-powered quality analysis.
- Analysis parameters.
- Damaged grains.
- Foreign matter.
- Grain uniformity.
- Visual condition.
- Moisture clues.
- Moisture warning.
- Recommendations.
- Estimated price range per quintal.
- Confidence:
  - High confidence
  - Medium confidence
  - Low confidence
- Milling recovery estimate.
- Rescan recommended.
- Scan failed state.
- Scan again action.

Photo tips:

- Spread grains on white paper or a plate.
- Keep a single layer with no overlapping grains.
- Use natural daylight or bright indoor light.
- Hold the camera 15-20 cm above the grains.
- At least 50 grains should be clearly visible.

Important availability note:

- The bundle also contains a modal saying "New Scanner Coming Soon" and says the Paddy Quality Scanner is being upgraded for better accuracy and will be back online shortly. This suggests scanner availability may vary.

### Crop Disease Detection

Function:

- Photo of Paddy, Cotton, or Chilli to detect disease and severity.
- Free detection is available.
- VIP gets full treatment with dosage.
- AI auto-identifies the crop and disease.
- Results in 15-30 seconds.

Inputs:

- Select crop to scan.
- Crop photo.
- Clear photo of affected leaves or stems.
- Photo any crop; AI identifies it automatically.

Outputs:

- Disease detected.
- No disease detected / crop looks healthy.
- Unknown crop.
- AI detected.
- Confidence.
- Symptoms observed.
- Cause.
- "Why did this happen?"
- Treatment.
- VIP treatment badge.
- Treatment locked message for non-VIP users.
- Scan disclaimer: AI visual assessment should be confirmed with a local agricultural officer for critical decisions.

Treatment data exposed in the bundle can include:

- What to do.
- Chemical treatment.
- Organic treatment.
- Best time to spray.
- Cost estimate.
- Exact dosage for field size for VIP users.

## Mandi Prices

Mandi pricing appears in public and VIP-related copy.

Visible claims:

- Live mandi prices.
- Real-time mandi prices.
- Today's live mandi rates for your district.
- Daily market rates from 100+ Telangana mandis.
- Helps farmers sell at the right price.

The homepage lists live mandi prices under the free plan, while some dashboard copy also says "Next Month: Prices and Voice" under VIP exclusive, so availability may depend on rollout status.

## Voice AI Call

Voice AI is presented as a VIP-exclusive feature.

Visible claims:

- Talk to Rythu AI in Telugu or English.
- Free plan: Voice AI Call 45-second free trial.
- Monthly subscription: Voice AI Call, 3 minutes per call.
- VIP exclusive Voice Assistant Telugu.
- "Next Month: Prices and Voice" appears in VIP copy.

The `/voice-test` route redirected to login, so the actual call UI could not be operated without a working browser login session.

The bundle includes microphone and real-time voice-session code, so the feature likely requires:

- Microphone permission.
- Browser voice/audio support.
- Logged-in account.
- VIP or trial access depending on plan.

## Profile And Account

Profile fields:

- Full name.
- Phone number.
- Village / Mandal.
- District.
- Email, read-only or linked account.
- Profile photo / camera icon.
- Save profile.
- Saved state.
- Save failed alert.

Account actions:

- Delete account.
- Confirmation before deletion.
- Account deleted state.
- Re-login required to delete account for security.

Location:

- GPS location is used for weather alerts.
- Weather screen has a location update action.

## VIP And Payments

Pricing shown on the homepage:

- Forever Free: Rs 0 forever.
- Monthly subscription: Rs 100/month.

Free plan includes:

- Daily crop tasks and reminders.
- 5-day weather forecasts.
- Access to government schemes.
- Live mandi prices.
- Voice AI Call 45-second free trial.

Monthly subscription includes:

- Unlimited AI chat support.
- Priority scheme applications.
- Lab-grade paddy scanner.
- Personal farming advisor.
- Voice AI Call, 3 minutes per call.

VIP benefits exposed in the app:

- Unlimited AI expert advice.
- Priority paddy scan analysis.
- No daily message limits.
- VIP farmer badge on profile.
- Unlimited chat and paddy scans with monthly pass.
- More crop tracking capacity.
- Full crop disease treatment with exact dosage.
- More eligibility matches in scheme checker.

Payment integration:

- Secure payment via Razorpay.
- Payment failed alert.
- Payment service unavailable alert.
- Payment not configured alert.
- Payment verification failed alert.
- VIP activated success alert.

## Privacy And Data Handling

The public privacy page says the app collects:

- Name, email, district, village.
- AI chat messages, with permission.
- Crop information: name, sowing date, acres.
- Scan history: disease and quality results.
- GPS location for weather alerts only.
- Payment ID only, not card details.

The privacy page says data is used for:

- Personalized farming advice.
- Daily crop reminders.
- Weather alerts.
- VIP payment processing.
- App performance improvement.

Third-party services listed:

- Google Gemini AI for chat messages and crop photos used for AI answers.
- Razorpay for payments.
- Resend for OTP and welcome emails.
- Firebase for profile and crop data.

The privacy page says the app does not store:

- Credit/debit card numbers.
- UPI IDs or bank details.
- Crisis message content, only metadata.
- Phone numbers, according to the privacy text.

User rights:

- Delete account from Settings/Profile.
- Clear chat history from chat page.
- Contact the app by email for data questions.

## Feature Workflow Matrix

This section explains each major feature in the same format: what it takes as input, how it works internally or logically, and what output the farmer receives.

### 1. Public Homepage And Onboarding

Inputs:

- User language choice: Telugu or English.
- User action: browse features, view pricing, click login, click get started, or click learn more.
- Cookie choice: accept or decline.

Logic / processing:

- The site presents the product value proposition and maps the user toward account creation.
- Feature cards explain what the app can do before login.
- Pricing cards separate free and paid capabilities.
- Navigation links route the user to login, pricing, or feature sections.

Outputs:

- Clear list of available farming features.
- Free and VIP plan comparison.
- Onboarding steps:
  - Create account.
  - Register crop.
  - Get AI help, weather alerts, and crop tasks.
- Login / signup entry point.

### 2. Authentication And Account Entry

Inputs:

- Email address.
- Password.
- Phone tab selection, if using phone login.
- Signup name, email, and password.
- Password reset email.
- Email verification status.

Logic / processing:

- Firebase Authentication validates login credentials.
- Signup creates a linked user account.
- Password reset sends a reset link.
- Email verification checks whether the user has clicked the verification email.
- Invalid credentials, weak passwords, missing users, or network failures return error messages.

Outputs:

- Logged-in session if authentication succeeds.
- Dashboard access after successful login and verification.
- Reset email sent message.
- Verification required message.
- Login errors such as invalid credential, wrong password, weak password, or network error.

### 3. Dashboard / Home

Inputs:

- Logged-in user profile.
- User name.
- Primary crop.
- District.
- VIP status and VIP expiry date.
- Saved location.
- Current date and time.

Logic / processing:

- The app personalizes the greeting by time of day.
- It shows the farmer's crop and district if profile data exists.
- It checks VIP expiry and warns when expiry is near.
- It fetches short weather and crop-task summaries when location and crop are available.
- It presents quick access cards to core features.

Outputs:

- Personalized welcome message.
- Farmer profile summary.
- Today's message or farming quote.
- VIP active / expiring / renewal notice.
- Weather summary.
- Today's crop task summary.
- Quick links to Learn, Schemes, Weather, Track Crop, Scan, and other tools.

### 4. AI Assistant

Inputs:

- Farmer question typed in chat.
- Optional crop/farm image.
- Optional speech-to-text input in Telugu or English.
- Recent chat history.
- Farmer context:
  - User ID.
  - Name.
  - District.
  - Village.
  - Language.
- Suggested prompt selected by the user.

Logic / processing:

- The app converts voice to text when speech recognition is used.
- The user message, optional image, chat history, and farmer context are sent to `/api/ai/chat`.
- Recent messages are included so the AI can answer with context.
- If an image is attached, it is sent as part of the AI request.
- The app retries once after a short delay if the first request fails.
- If the request takes too long, it returns a busy/server error.
- The app detects daily-limit or upgrade-related replies and can trigger an upgrade prompt.

Outputs:

- AI farming advice in the selected language.
- Crop, disease, fertilizer, organic farming, and scheme-related answers.
- Image-based diagnosis response when a photo is attached.
- Chat history saved.
- Audio playback if sound is enabled.
- Error output if the server is busy or network fails.

### 5. Weather Alerts

Inputs:

- Latitude.
- Longitude.
- Location name.
- Number of forecast days.
- Selected forecast day.
- Crop-specific weather alert toggle.

Logic / processing:

- The app calls `/api/weather/forecast?lat=<latitude>&lon=<longitude>&days=5`.
- It reads daily temperature, rain probability, weather code, wind speed, sunrise, and sunset.
- It reads hourly humidity and uses the midday value for the selected day.
- It converts weather codes into readable labels and icons.
- It creates farm tips using thresholds:
  - Rain chance above 60 percent means delay harvesting or drying.
  - Temperature above 38 C means ensure irrigation.
  - Otherwise the weather is treated as good for farm activities.
- Crop tracker uses rain and heat alerts to warn about fertilizer, pesticide, and irrigation timing.

Outputs:

- Today's or selected day's weather.
- Weather condition name.
- Max and min temperature.
- Rain chance.
- Humidity.
- Wind speed.
- Feels-like temperature.
- Sunrise time.
- Sunset time.
- 5-day forecast list.
- Farming tip for the selected day.
- Retry/error message if weather cannot load.
- Crop-specific weather warnings.

### 6. Farming Guide / Learn

Inputs:

- Selected crop category:
  - Vegetables.
  - Leafy Vegetables.
  - Kharif Crops.
  - Rabi Crops.
- Selected crop.
- Language preference.

Logic / processing:

- The app loads static crop-guide data from the frontend bundle.
- It groups crops by sector/category.
- It displays crop-specific agronomy details in English or Telugu.
- Each crop guide combines soil, season, duration, irrigation, fertilizer, expected yield, disease/cure, and tips.

Outputs:

- Crop overview.
- Soil type guidance.
- Best season.
- Crop duration.
- Irrigation schedule.
- Fertilizer schedule.
- Expected yield.
- Disease and cure information.
- Pro tips.
- Telugu/English crop guidance.

### 7. Track Crop

Inputs:

- Crop name.
- Sowing date.
- Acres / land size.
- Location.
- GPS detected location.
- Weather alerts toggle.
- Daily email reminders toggle.
- Crop stage selected by the farmer.
- Crop notes.
- Note type:
  - Observation.
  - Issue.
  - Action.
- Note priority:
  - Urgent.
  - Important.
  - Info.
- VIP status.

Logic / processing:

- The app creates a tracked crop record from crop name, sowing date, land size, and location.
- It uses sowing date to calculate crop day and total crop progress.
- The farmer can update stage from planning through harvested.
- It checks crop count limits:
  - Free users can track up to 2 crops.
  - VIP users can track up to 5 crops.
- Weather and heat alerts are connected to crop timing if enabled.
- It fetches today's crop task from `/crop/today-task?uid=<user_id>`.
- It shows premium AI insights only for VIP or upgrade-eligible users.

Outputs:

- Crop tracking card.
- Current crop stage.
- Day number and total days.
- Today's task.
- Heavy rain / heat alert.
- Notes history.
- AI crop insights.
- Fertilizer schedule and pest warnings for premium users.
- Mark harvested action.
- Remove crop action.
- Upgrade prompt if free crop limit is reached.

### 8. Government Schemes

Inputs:

- Farmer profile and location.
- Land ownership.
- Land-size category.
- Age.
- Gender.
- Category/caste group.
- Annual income tier.
- Occupation.
- Document availability:
  - Aadhaar Card.
  - Bank Account.
  - Kisan Credit Card.
  - Crop Insurance.
  - Land Records / Pattadar Passbook.

Logic / processing:

- The app compares farmer details with each scheme's eligibility rules.
- It checks whether the farmer is land owner, tenant, marginal, small, or large.
- It uses income and occupation to exclude farmers from schemes where needed.
- It warns that government employees and income tax payers may not qualify for many schemes.
- It checks document readiness to decide whether the farmer can apply or must collect documents.
- It separates eligible and not-eligible schemes.

Outputs:

- Eligible schemes.
- Not eligible schemes.
- Estimated total benefit.
- Scheme-wise benefit amount.
- Why the farmer is eligible.
- Reason for non-eligibility.
- Required documents.
- How to apply.
- What to do to qualify.
- Helpline.
- Official website.
- Apply online link.
- VIP prompt for more matching schemes if applicable.

### 9. Paddy Quality Scanner

Inputs:

- Paddy variety.
- Paddy photo.
- Photo quality:
  - Grains visible.
  - Single layer.
  - Good lighting.
  - Camera distance.
  - File size under 1 MB.
- VIP status.

Logic / processing:

- The app accepts or captures a paddy image.
- It checks whether a photo is ready and whether file size is valid.
- It sends the image for AI visual quality analysis.
- It estimates grain quality indicators from the image.
- It can recommend a rescan when quality/confidence is low.
- The feature may show a "coming soon / upgrading" modal depending on current rollout status.

Outputs:

- Paddy grade such as Grade A/B/C/D.
- Paddy quality analysis.
- Damaged grains.
- Foreign matter.
- Grain uniformity.
- Visual condition.
- Moisture clues.
- Moisture warning.
- Estimated price range per quintal.
- Milling recovery estimate.
- Confidence level.
- Recommendations.
- Retake tips.
- Scan failed / scan again state.
- VIP upsell if scanner access is restricted.

### 10. Crop Disease Detection

Inputs:

- Crop photo.
- Selected crop, if manually selected.
- Target crops mentioned in app copy:
  - Paddy.
  - Cotton.
  - Chilli.
- Photo of affected leaves or stems.
- VIP status.

Logic / processing:

- The app uploads the crop photo for AI disease detection.
- AI attempts to identify crop and disease automatically.
- It checks visible symptoms and disease severity.
- Free users receive detection.
- VIP users receive full treatment details and dosage.
- It shows a disclaimer that AI visual assessment should be confirmed with a local agriculture officer for critical decisions.

Outputs:

- Disease detected / no disease detected.
- Unknown crop result if AI cannot identify.
- AI detected label.
- Confidence score.
- Symptoms observed.
- Cause.
- Explanation of why it happened.
- Treatment summary.
- Chemical treatment.
- Organic treatment.
- Best spray time.
- Cost estimate.
- Exact dosage for VIP users.
- Scan failed / scan again state.

### 11. Mandi Prices

Inputs:

- Farmer district.
- Crop or commodity context.
- Current date.
- Market/mandi data source from the backend.
- VIP or rollout status, depending on availability.

Logic / processing:

- The app is designed to fetch or show live mandi rates for the farmer's district.
- It positions the data as daily market rates from Telangana mandis.
- It helps compare current market price with expected selling value.
- Some copy says live mandi prices are free, while other copy says "Next Month: Prices and Voice," so this feature may be partially rolled out.

Outputs:

- Today's mandi rates.
- District-level price guidance.
- Crop/commodity price information.
- Selling-at-right-price guidance.
- Feature unavailable / coming soon state if not active.

### 12. Voice AI Call

Inputs:

- Microphone permission.
- Spoken Telugu or English.
- Logged-in user account.
- VIP status or free-trial eligibility.
- Call duration limit:
  - Free trial: 45 seconds.
  - VIP: 3 minutes per call.

Logic / processing:

- The route requires login.
- The app checks account and access level.
- It uses microphone/audio APIs for real-time voice conversation.
- It connects the user to the Rythu Voice AI session.
- The feature appears linked to VIP or upcoming rollout copy.

Outputs:

- Spoken conversation with Rythu AI.
- Telugu/English voice farming support.
- Trial or VIP call session.
- Microphone permission errors if denied.
- Login redirect if unauthenticated.
- Access/upgrade prompt if not eligible.

### 13. Profile And Account

Inputs:

- Full name.
- Phone number.
- Village / Mandal.
- District.
- Email from linked account.
- Profile photo.
- Delete account confirmation.

Logic / processing:

- The app saves profile fields to the user's Firebase-backed profile.
- Email is linked/read-only.
- Profile data can auto-fill other flows such as schemes and dashboard.
- Profile photo can be changed through camera/photo upload.
- Account deletion requires confirmation and may require recent re-login.

Outputs:

- Saved farmer profile.
- Dashboard personalization.
- District/village context for AI and schemes.
- Profile photo update.
- Save success / save failed message.
- Account deleted message.
- Re-login required message for secure deletion.

### 14. VIP And Payments

Inputs:

- Selected plan.
- User account.
- Payment action.
- Razorpay payment response.
- Payment ID / verification data.
- Current VIP expiry date.

Logic / processing:

- The app starts a Razorpay payment flow for VIP.
- It verifies payment using backend payment verification.
- If payment succeeds, VIP is activated.
- If VIP is active, premium feature limits are unlocked.
- If VIP expiry is near, the app shows renewal warnings.
- Payment failures and unavailable payment service states are handled with alerts.

Outputs:

- VIP activated success message.
- Active membership status.
- Membership expired status.
- VIP expiry warning.
- Renew now action.
- Unlimited AI chat support.
- Priority paddy scan analysis.
- More crop tracking.
- Full disease treatment dosage.
- More matching schemes.
- Payment failed / verification failed / payment unavailable errors.

### 15. Privacy And Data Handling

Inputs:

- User profile data.
- AI chat messages.
- Crop details.
- Scan photos and results.
- GPS location for weather alerts.
- Payment ID.
- User deletion or clear-history request.

Logic / processing:

- Firebase stores profile and crop data.
- Google Gemini receives chat messages and crop photos for AI answers.
- Razorpay processes payments.
- Resend sends OTP and welcome emails.
- GPS is used for weather alerts.
- The app says it stores payment IDs, not card/UPI/bank details.
- Clear history removes chat history.
- Delete account removes user data through profile settings.

Outputs:

- Personalized farming service.
- AI chat and scan answers.
- Weather alerts.
- Crop reminders.
- VIP payment status.
- Clear chat history result.
- Delete account result.
- Privacy contact path through email.

## Feature Availability Summary

Directly visible without login:

- Landing page.
- Feature descriptions.
- Pricing.
- Login/signup/reset shell.
- Privacy policy.

Requires login:

- Dashboard.
- AI assistant.
- Weather screen.
- Crop tracker.
- Crop guide screen.
- Scheme checker.
- Scan lab.
- Profile.
- Payments/VIP.
- Voice AI route.

Could not be operated because browser login failed:

- Authenticated dashboard interactions.
- Live AI chat request.
- Live weather query from saved location.
- Crop tracker creation.
- Scheme eligibility form submission.
- Paddy scanner upload.
- Disease scanner upload.
- Voice AI call.
- Razorpay upgrade flow.
