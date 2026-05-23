# Backend Setup — Google Sheets + Google Apps Script

This document explains how to connect the form to a Google Sheet for storing responses and feeding the community progress bar.

## Architecture

```
Landing (GitHub Pages)
    │
    ├── POST / → Google Apps Script → Google Sheets (responses)
    │
    └── GET ?action=counts → Google Apps Script → returns { musicians, bands }
```

The landing is pure static HTML/JS on GitHub Pages. Google Apps Script acts as the backend.

---

## Step 1 — Create the Google Sheet

1. Go to [sheets.new](https://sheets.new)
2. Create columns with these exact headers in row 1:

### Common fields (all profiles)
```
timestamp | profileType | email | name | cityZone | ageRange | alternativeContact
```

### Individual musician fields
```
mainInstrument | secondaryInstruments | level | preferredStyles | musicReferences | primaryGoal | futureInterests | commitment | availability | desiredFrequency | currentlyPlaying | searchedBefore | searchChannels | rehearsedBefore | biggestChallenge | matchPriorities | trustFactors | goodExperience
```

### Band/group fields
```
bandName | bandMembersCount | bandCurrentInstruments | bandLookingForInstruments | bandStage | bandStyles | bandReferences | bandLevel | bandMainGoal | bandFutureInterests | commitment | availability | desiredFrequency | bandSearchedBefore | bandSearchChannels | bandRehearsedBefore | bandBiggestChallenge | bandMatchPriorities | bandTrustFactors | bandGoodExperience
```

### Consent fields
```
researchParticipation | consent
```

3. Copy the **Sheet ID** from the URL: `https://docs.google.com/spreadsheets/d/{SHEET_ID}/edit`

---

## Step 2 — Create the Apps Script

1. In your Sheet, go to **Extensions > Apps Script**
2. Delete any default code and paste the script below
3. Replace `YOUR_SHEET_ID_HERE` with your actual Sheet ID
4. Save the project (name it e.g. "FormMusicBackend")

```javascript
const SHEET_ID = 'YOUR_SHEET_ID_HERE';

function doPost(e) {
  try {
    const sheet = SpreadsheetApp.openById(SHEET_ID).getActiveSheet();
    const data = JSON.parse(e.postData.contents);

    const row = [
      new Date().toISOString(),                                              // timestamp
      data.profileType || '',                                                // profileType
      data.email || '',                                                       // email
      data.name || '',                                                        // name
      data.cityZone || '',                                                    // cityZone
      data.ageRange || '',                                                    // ageRange
      data.alternativeContact || '',                                          // alternativeContact
    ];

    if (data.profileType === 'band') {
      row.push(
        data.bandName || '',
        data.bandMembersCount || '',
        (data.bandCurrentInstruments || []).join(', '),
        (data.bandLookingForInstruments || []).join(', '),
        data.bandStage || '',
        (data.bandStyles || []).join(', '),
        data.bandReferences || '',
        data.bandLevel || '',
        data.bandMainGoal || '',
        (data.bandFutureInterests || []).join(', '),
        data.bandCommitment || '',
        data.availability || '',
        data.bandDesiredFrequency || '',
        data.bandSearchedBefore || '',
        (data.bandSearchChannels || []).join(', '),
        data.bandRehearsedBefore || '',
        data.bandBiggestChallenge || '',
        (data.bandMatchPriorities || []).join(', '),
        (data.bandTrustFactors || []).join(', '),
        data.bandGoodExperience || ''
      );
    } else {
      row.push(
        data.mainInstrument || '',
        (data.secondaryInstruments || []).join(', '),
        data.level || '',
        (data.preferredStyles || []).join(', '),
        data.musicReferences || '',
        data.primaryGoal || '',
        (data.futureInterests || []).join(', '),
        data.commitment || '',
        data.availability || '',
        data.desiredFrequency || '',
        data.currentlyPlaying || '',
        data.searchedBefore || '',
        (data.searchChannels || []).join(', '),
        data.rehearsedBefore || '',
        data.biggestChallenge || '',
        (data.matchPriorities || []).join(', '),
        (data.trustFactors || []).join(', '),
        data.goodExperience || ''
      );
    }

    // Consent fields
    row.push(
      data.researchParticipation || '',
      data.consent || ''
    );

    sheet.appendRow(row);
    return ContentService
      .createTextOutput(JSON.stringify({ success: true }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ success: false, error: err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function doGet(e) {
  try {
    const sheet = SpreadsheetApp.openById(SHEET_ID).getActiveSheet();
    const rows = sheet.getDataRange().getValues();

    let musicians = 0;
    let bands = 0;

    // Skip header row (index 0)
    for (let i = 1; i < rows.length; i++) {
      const profileType = rows[i][1]; // column B = profileType
      if (profileType === 'band') {
        bands++;
      } else if (profileType === 'individual') {
        musicians++;
      }
    }

    return ContentService
      .createTextOutput(JSON.stringify({
        musicians: musicians,
        bands: bands,
        targets: { musicians: 500, bands: 100 }
      }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ error: err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

---

## Step 3 — Deploy as Web App

1. In the Apps Script editor, click **Deploy > New deployment**
2. Choose type: **Web app**
3. Configure:
   - **Description**: e.g. "FormMusic Backend"
   - **Execute as**: "Me" (uses your Google account)
   - **Who has access**: "Anyone" (the form needs public access)
4. Click **Deploy**
5. **Copy the Web App URL** — it looks like:
   ```
   https://script.google.com/macros/s/{SCRIPT_ID}/exec
   ```

---

## Step 4 — Connect the Frontend

1. Open `index.html`
2. Find the `CONFIG` object at the top of the `<script>` section
3. Replace the placeholder URL:

```javascript
const CONFIG = {
    SHEETS_ENDPOINT: "https://script.google.com/macros/s/YOUR_SCRIPT_ID/exec",
    COMMUNITY_TARGETS: { musicians: 500, bands: 100 }
};
```

---

## Step 5 — Test

1. Open the landing page in a browser
2. Fill out and submit the form
3. Check the Google Sheet — a new row should appear
4. The progress bar should update after page reload
5. Test the counts endpoint by visiting:
   ```
   https://script.google.com/macros/s/YOUR_SCRIPT_ID/exec?action=counts
   ```
   Response should look like:
   ```json
   { "musicians": 3, "bands": 1, "targets": { "musicians": 500, "bands": 100 } }
   ```

---

## Security & Privacy

- The `doGet` function **only returns aggregate counts** — no emails, names, or personal data.
- The spreadsheet is accessible only to you (the Sheet owner) and anyone you explicitly share it with.
- No credentials or tokens are committed to the repository — the URL is a public Apps Script endpoint.
- The endpoint only accepts POST requests with JSON bodies; it validates nothing further — add validation inside `doPost` if needed.

---

## Updating Targets

To change the community goals (e.g., 500 → 1000):

1. Update `CONFIG.COMMUNITY_TARGETS` in `index.html`
2. Update the `targets` object in the Apps Script `doGet` function
3. Update the displayed numbers in the progress bar HTML if needed
