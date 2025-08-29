# Chrome Extension – Lead Collection Documentation

---

## Overview

This Chrome Extension collects real estate leads from three supported platforms:

* **iSpeedToLead** (`https://app.ispeedtolead.com/*`)
* **Motivated Sellers** (`https://motivatedsellers.com/*`)
* **Real Estate Bees** (`https://dashboard.realestatebees.com/*`)

All captured leads are sent to the backend API for processing.

---

## Backend API Endpoint

```
POST {base_url}/api/ispeed_properties/lead
```

**Body:**

```json
{
  "lead": {...},
  "source": "website-domain"
}
```

---

## Lead Collection Flow

### 1. iSpeedToLead

* Trigger: User clicks on **eye (View Sold Comps)** button.
* Action: Extension scrapes lead details + sold comps.
* Example Payload:

```json
{
  "id": "68829953049df8ccc9343c02",
  "status": "SALE LEAD",
  "date": "24 July 2025, 04:38 PM",
  "orders": "1 orders",
  "provider": "Sell Your House Fast",
  "verified": "Phone Number Verified",
  "populations": [
    { "title": "City:", "value": "911507" },
    { "title": "County:", "value": "995708" }
  ],
  "details": {
    "ZIP code": "32246",
    "Type of Property": "Single family",
    "Square footage": "1000 - 2000"
  },
  "has_sold_comps": true,
  "sold_comps_opened": true,
  "sold_comps": [
    {
      "address": "2733 Lantana Lakes Dr E, Jacksonville, FL 32246",
      "sale_price": "$430,000",
      "beds": "3",
      "baths": "2",
      "sqft": "1,874"
    }
  ],
  "source": "ispeedtolead.com"
}
```

---

### 2. Real Estate Bees

* Trigger: User clicks **See More Details**.
* Action: Extension scrapes full lead details.
* Example Payload:

```json
{
  "lead": [
    {
      "complete_lead": true,
      "lead_type": "Direct Lead",
      "phone_verified": false,
      "request_type": "Cash Offer",
      "property_type": "House",
      "location": "Jacksonville, FL 32234",
      "lead_details": {
        "Lead submitted": "06/10/2025 11:24am",
        "Desired asking price": "$300,000",
        "Current mortgage/debt": "$150,000",
        "Reason for selling": "Investing in another business venture",
        "Beds": "4",
        "Baths": "3"
      }
    }
  ],
  "source": "realestatebees.com"
}
```

---

### 3. Motivated Sellers

* Trigger: On **page load**, all visible leads are scraped.
* Action: Send lead list to backend.
* Example Payload:

```json
{
  "lead": [
    {
      "name": "Jo*** Es*****",
      "phone": "682*******",
      "email": "je******@gm***.com",
      "address": "164* ********* **",
      "city": "Grand Prairie",
      "state": "TX",
      "zipcode": "75051",
      "county": "DALLAS",
      "received": "Jul/24/25",
      "moreInfoUrl": "https://motivatedsellers.com/leads/app/preview/4ab6e965"
    }
  ],
  "source": "motivatedsellers.com"
}
```

---

## Email Monitoring

For **Motivated Sellers** and **Real Estate Bees**:

* Extension checks frequently for new lead emails.
* If a new email is detected:

  * Open the lead link.
  * Log in (if required).
  * Scrape details.
  * Send payload to backend.

---



# Backend – Lead Processing & AI Calling Flow

---

## 1. Lead Intake

* API receives lead:

  ```
  POST {base_url}/api/ispeed_properties/lead
  ```
* Determine source:

  * `ispeedtolead.com`
  * `realestatebees.com`
  * `motivatedsellers.com`

Process according to source logic.

---

## 2. Source-Specific Processing

### iSpeedToLead

1. Save lead details in DB.
2. Save sold comps in DB.
3. Process sold comps:

   * Extract property details & locations.
   * Update DB with location data.
   * Generate circles and find **most interactive point**.
   * Pick properties from that point.
   * Match against lead details → update exact property in DB.
4. If property already exists but comps missing:

   * Fetch comps.
   * Re-run process.

---

### RealEstateBees

1. Save lead details in DB.
2. Find property using:

   * City
   * State
   * County
   * Zip
   * Bedrooms / Bathrooms
   * Building Area
   * Lot Size
3. Match exact property → update in DB.

---

### Motivated Sellers

1. Save lead details in DB.
2. Match property using:

   * City, State, County, Zip.
3. Apply **address pattern matching**.
4. Validate owner:

   * Compare first characters of first + last name.
5. If matched → update property in DB.

---

## 3. Property Enrichment Flow

Once property **address is found**:

1. Add address into DB.
2. Send address → **DealMachine** (for owner details).
3. After **15 seconds**, use GET request with property address → fetch owner + phones from DealMachine.
4. Update owners + phone numbers in DB.

---

## 4. Phone Enrichment

1. Use **Scrappy**:

   * Search phone numbers by address + owner name.
   * Add found numbers into DB.
2. Run **EDQ check**:

   * Validate number status:

     * Blocked
     * Working
     * Type (Mobile / Landline)
   * Remove blocked numbers.
   * Keep valid numbers for AI calling.

---

## 5. AI Calling Flow

1. AI Agent calls valid numbers.
2. Outcomes:

   * **Confirmed Owner** → stop calling other numbers.
   * **Not Confirmed** → mark as wrong number.
   * **Voicemail** → add to **AI Schedule Calling**.

---

## 6. AI Schedule Calling (Voicemail Numbers)

* Day 1 → 3 calls
* Day 2 → 2 calls
* Days 3–14 → 1 call per day
* All calls respect phone **timezone window: 12 PM – 6 PM**
* If at any point:

  * **User confirmed / Not confirmed** → stop other scheduled calls.
* If voicemail every time → mark status: **Continues Voicemailed**.

---

## 7. CRM + Dialer Integration

* Once owner is confirmed:

  * Add/Update contact into **GHL** (GoHighLevel).
  * Add contact into **Kixie**.

---

## 8. Fast-Track Scenario

* If property already has **address, owner name, and phone numbers**:

  * Skip DealMachine + Scrappy steps.
  * Continue directly with EDQ → AI Calling → CRM & Dialer.

---



