# API Ethics Assignment - Healthcare Data Handling

**Name:** Kirubakar  
**Role:** Junior Data Analyst  
**Date:** April 01, 2026

## Task 1 — Classify and Handle PII Fields

### Classification and Recommended Handling:

| Field              | Type of PII                  | Recommended Action      | Justification |
|--------------------|------------------------------|-------------------------|-------------|
| **full_name**      | Direct PII                   | **Drop**                | Directly identifies the individual. No research value that cannot be achieved through other means. |
| **email**          | Direct PII                   | **Drop**                | Strong personal identifier. High risk of re-identification and potential for misuse. |
| **date_of_birth**  | Direct PII                   | **Pseudonymize**        | Convert to age or age groups (e.g., 25-34). Retains useful analytical value while reducing risk. |
| **zip_code**       | Indirect PII                 | **Mask / Generalize**   | Generalize to first 3 digits (e.g., 600XX) or city level to prevent precise location tracking. |
| **job_title**      | Indirect PII                 | **Keep**                | Not highly sensitive. Useful for demographic and socioeconomic analysis in healthcare research. |
| **diagnosis_notes**| Sensitive Health Data (PHI) | **Pseudonymize + Review**| Contains medical information. Should be generalized (e.g., specific disease → broader category) or reviewed carefully before sharing. |

**Summary Recommendation:**  
Drop `full_name` and `email`. Pseudonymize `date_of_birth` and carefully process `diagnosis_notes`. Generalize `zip_code`. Keep `job_title`.

---

## Task 2 — Audit the API Script for Ethical Compliance

### Violation 1: Hardcoded API Key (Security Risk)

**Problem:**  
The API key is hardcoded directly in the script. This is insecure because anyone who sees the code (colleagues, GitHub, etc.) can access and potentially misuse the key. It also violates secure credential management practices.

**Corrected Code:**

```python
import requests
import os

API_URL = "https://healthstats-api.example.com/records"

# Load API key securely from environment variable
API_KEY = os.getenv("HEALTH_API_KEY")

if not API_KEY:
    raise ValueError("HEALTH_API_KEY environment variable is not set")

records = []
for page in range(1, 101):
    response = requests.get(API_URL, params={"page": page, "key": API_KEY})
    data = response.json()
    records.extend(data["results"])

save_to_database(records)

### Violation 2: No Rate Limiting and Potential API Abuse

**Problem:**  
The script loops through 100 pages rapidly without any delay between requests. This can overload the API server, violate the provider’s Terms of Service (TOS), and risk getting the API key banned or blocked. Responsible data collection requires respecting rate limits.

**Corrected Code:**

```python
import requests
import time
import os

API_URL = "https://healthstats-api.example.com/records"
API_KEY = os.getenv("HEALTH_API_KEY")

if not API_KEY:
    raise ValueError("HEALTH_API_KEY environment variable is not set")

records = []
max_pages = 100

for page in range(1, max_pages + 1):
    try:
        response = requests.get(
            API_URL, 
            params={"page": page, "key": API_KEY},
            timeout=10
        )
        
        response.raise_for_status()
        data = response.json()
        
        if not data.get("results"):
            break  # Stop when no more results
            
        records.extend(data["results"])
        print(f"Fetched page {page} - {len(data['results'])} records")
        
        time.sleep(1.0)  # 1 second delay to respect rate limits
        
    except requests.exceptions.HTTPError as e:
        if response.status_code == 429:   # Too Many Requests
            print("Rate limit reached. Waiting longer...")
            time.sleep(5)
        else:
            print(f"HTTP Error on page {page}: {e}")
            break
    except Exception as e:
        print(f"Error on page {page}: {e}")
        break

save_to_database(records)
print(f"Total records collected: {len(records)}")
