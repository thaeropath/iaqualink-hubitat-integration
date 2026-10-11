# iAqualink Hubitat Driver

A Hubitat Elevation device driver for the iAqualink pool control system.

**Original driver by [Vyrolan](https://github.com/Vyrolan/iAqualink-Hubitat).** This fork adds a Google Sheets temperature logging integration by [thaeropath](https://github.com/thaeropath). All credit for the core driver belongs to Vyrolan.

---

## Features

- Monitors pool/spa temperatures, heater states, and auxiliary equipment
- Creates child devices for temperature sensors, heaters, and AUX controls
- Auto-updates on a configurable interval
- **Google Sheets integration** — logs pool and/or spa temperature readings to a Google Sheet on every update cycle, with automatic spa mode awareness

---

## Google Sheets Integration Setup

### Prerequisites

- Google account with Google Sheets and Google Apps Script access
- At least one of **Enable Pool Temp Sensor** or **Enable Spa Temp Sensor** turned on in driver preferences

### Step 1 — Create the Google Sheet

1. Go to [Google Sheets](https://sheets.google.com) and create a new spreadsheet.
2. In row 1, add column headers: `Timestamp`, `Pool Temp`, `Spa Temp`, `Unit`, `Device`.

### Step 2 — Create the Apps Script Web App

1. In your Google Sheet, click **Extensions → Apps Script**.
2. Delete any existing content and paste the following:

```javascript
function doPost(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var data = JSON.parse(e.postData.contents);
  sheet.appendRow([
    data.timestamp,
    data.pool_temp !== undefined ? data.pool_temp : "",
    data.spa_temp  !== undefined ? data.spa_temp  : "",
    data.unit,
    data.device_label
  ]);
  return ContentService.createTextOutput("OK");
}
```

3. Click **Save** (name the project anything you like).

### Step 3 — Deploy the Web App

1. Click **Deploy → New Deployment**.
2. Click the gear icon next to "Type" and select **Web App**.
3. Set the following:
   - **Description**: anything (e.g. `iAqualink Logger`)
   - **Execute as**: Me
   - **Who has access**: Anyone
4. Click **Deploy**.
5. Copy the **Web App URL** — it will look like `https://script.google.com/macros/s/XXXXX/exec`.

> **Note:** If you edit the Apps Script later, you must create a **New Deployment** each time. Re-deploying updates the URL, so update the driver preference accordingly.

### Step 4 — Configure the Hubitat Driver

1. Open your iAqualink device in the Hubitat web UI.
2. Under **Preferences → Pool/Spa Settings**, enable at least one of:
   - **Enable Pool Temp Sensor** — required for pool temp logging
   - **Enable Spa Temp Sensor** — required for spa temp logging
3. Scroll to the **Google Sheets Integration** section and:
   - Toggle **Enable Google Sheets Temperature Logging** to on.
   - Paste the Web App URL from Step 3 into **Apps Script Web App URL**.
   - Optionally enable **Skip Logging When Filter Pump is Off**.
4. Click **Save Preferences**.

### What Gets Logged

Each update cycle appends one row to the sheet. Which temperature columns are populated depends on your system type and current mode:

| System Type | Pool Temp column | Spa Temp column |
|---|---|---|
| Pool-only | Current pool temp | *(blank)* |
| Spa-only | *(blank)* | Current spa temp |
| Pool+Spa in pool mode | Current pool temp | *(blank)* |
| Pool+Spa in spa mode | *(blank)* | Current spa temp |

**Example rows:**

| Timestamp | Pool Temp | Spa Temp | Unit | Device |
|---|---|---|---|---|
| 2026-04-20T08:30:00-07:00 | 78 | | F | Pool Controller |
| 2026-04-20T09:00:00-07:00 | | 102 | F | Pool Controller |

When spa mode is active, pool temp is automatically skipped because the pump is circulating spa water — the pool temp reading is unreliable in that state.

---

## Salt Cell Weather Automation

Sun burns off chlorine faster, so the driver can run the salt cell (SWC) harder on sunny days and ease off when it's cloudy.

### Setup

Under **Preferences → Salt Cell Weather Automation**:

| Preference | Default | Meaning |
|---|---|---|
| Enable Weather-Based Salt Cell Output | off | Turns the automation on |
| Sunny Weather Output (%) | 60 | Pool output used when it's sunny |
| Cloudy Weather Output (%) | 30 | Pool output used when it's cloudy or night |
| Cloudy Threshold (% Cloud Cover) | 60 | Cloud cover at or above this counts as cloudy |

### How it works

- Each update cycle, the driver fetches current cloud cover for the hub's location from [Open-Meteo](https://open-meteo.com) (free, no API key). Set the hub's location under **Settings → Location and Modes**.
- Night always counts as cloudy.
- To avoid flapping, once it's cloudy, cloud cover must drop 10 points below the threshold before it switches back to sunny.
- The output is only written when it differs from the current setting. The spa output is left unchanged, and nothing is changed while Boost is active.

### Commands and attributes

- **setWeatherCondition(auto / sunny / cloudy)**: overrides Open-Meteo, for example from a Rule Machine rule that uses your own weather device. `auto` resumes Open-Meteo.
- **setSaltOutput(percent)**: sets the pool salt cell output manually.
- Attributes: `swc_pool_output`, `swc_spa_output`, `cloud_cover`, `weather_condition`.

> **Note:** iAqualink doesn't publicly document the command that sets salt cell output. The driver uses `set_swc_config`, logs the controller's response, and re-reads the output afterwards. If you see a warning that the output didn't change, please open an issue with the logged response.

---

## Credits

- **[Vyrolan](https://github.com/Vyrolan)** — original iAqualink Hubitat driver
- **[thaeropath](https://github.com/thaeropath)** — Google Sheets integration
