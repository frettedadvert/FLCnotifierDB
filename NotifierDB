import requests
import yagmail
import json
import os
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager

os.environ['TF_CPP_MIN_LOG_LEVEL'] = '3'  # Suppress TensorFlow logging

websites = [
    {"url": "https://bieterportal.noncd.db.de/evergabe.bieter/eva/supplierportal/portal/tabs/vergaben", "keywords": ["catering", "verpflegung", "lebensmittel", "kantin", "speise", "hotel", "essen"]},
]

# Email configuration
EMAIL_ADDRESS = os.getenv("EMAIL_ADDRESS")
EMAIL_PASSWORD = os.getenv("EMAIL_PASSWORD")

SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
MATCHES_FILE = os.path.join(SCRIPT_DIR, "matches.json")
TEXT_PARTS_FILE = "extracted_text_parts.json"

def load_previous_matches():
    """Load previously found matches from a file."""
    if os.path.exists(MATCHES_FILE):
        with open(MATCHES_FILE, "r") as file:
            return json.load(file)
    return {}

def save_matches(matches):
    """Save matches to a file."""
    with open(MATCHES_FILE, "w") as file:
        json.dump(matches, file, indent=4)

def save_text_parts(text_parts):
    """Save extracted text parts to a file."""
    with open(TEXT_PARTS_FILE, "w") as file:
        json.dump(text_parts, file, indent=4)

def search_keywords(extracted_data, keywords):
    """Check extracted titles for keyword matches."""
    relevant_matches = []
    for data in extracted_data:
        title = data["title"].lower()
        date = data.get("date", "No Date")
        
        if any(keyword.lower() in title for keyword in keywords):
            print(f"Match found: {title}")
            relevant_matches.append({"title": data["title"], "date": date})
    return relevant_matches

def extract_titles_with_selenium(url):
    """Extract titles and corresponding dates from a dynamically rendered webpage using Selenium."""
    extracted_data = []
    
    chrome_options = Options()
    chrome_options.add_argument("--headless")
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-dev-shm-usage")
    chrome_options.add_argument("--window-size=1920,1080")
    chrome_options.add_argument("--disable-blink-features=AutomationControlled")
    
    service = Service(ChromeDriverManager().install())
    driver = webdriver.Chrome(service=service, options=chrome_options)
    
    try:
        driver.get(url)
        try:
            WebDriverWait(driver, 30).until(
                EC.element_to_be_clickable((By.XPATH, "//button[contains(text(), 'alle akzeptieren')]"))
            ).click()
        except Exception:
            pass

        wait = WebDriverWait(driver, 30)
        title_elements = wait.until(EC.presence_of_all_elements_located((By.CLASS_NAME, "flex-basis-90")))
        date_elements = wait.until(EC.presence_of_all_elements_located((By.CLASS_NAME, "ng-star-inserted")))

        date_elements = [date_element for date_element in date_elements if len(date_element.text.strip()) == 15]

        for title_element, date_element in zip(title_elements, date_elements):
            title = title_element.text.strip() or title_element.get_attribute("innerText").strip()
            date = date_element.text.strip()
            extracted_data.append({"title": title, "date": date})
    
    finally:
        driver.quit()
    
    return extracted_data

def send_email(new_matches):
    """Send an email notification with titles and dates."""
    subject = "Neue Ausschreibungen verfügbar!!"
    body = "Die folgenden neuen Übereinstimmungen wurden gefunden:\n"
    body += "URL: https://bieterportal.noncd.db.de/evergabe.bieter/eva/supplierportal/portal/tabs/vergaben\n\n"
    
    for match in new_matches:
        title = match.get("title", "No Title")
        date = match.get("date", "No Date")
        body += f"Title: {title}\nDeadline: {date}\n\n"
    
    try:
        yag = yagmail.SMTP(EMAIL_ADDRESS, EMAIL_PASSWORD)
        yag.send("Henrik.Hemmer@flc-group.de", subject, body)
        print("Email sent!")
    except Exception as e:
        print(f"Failed to send email: {e}")

def main():
    """Main function to check websites and send emails."""
    previous_matches = load_previous_matches()
    new_matches = []

    for site in websites:
        url = site["url"]
        keywords = site["keywords"]

        extracted_data = extract_titles_with_selenium(url)
        save_text_parts(extracted_data)

        matches = search_keywords(extracted_data, keywords)
        if url not in previous_matches:
            previous_matches[url] = []

        for match in matches:
            if match not in previous_matches[url]:
                new_matches.append(match)
                previous_matches[url].append(match)

    if new_matches:
        send_email(new_matches)
    
    save_matches(previous_matches)

if __name__ == "__main__":
    main()
