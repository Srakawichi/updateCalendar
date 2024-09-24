# updateCalendar
Updates Google calender if a change is detected

Import library: pip install requests icalendar google-auth google-auth-oauthlib google-auth-httplib2 google-api-python-client

2. Erstelle ein Google Cloud-Projekt und aktiviere die Google Calendar API
Gehe zur Google Cloud Console.
Erstelle ein neues Projekt.
Aktiviere die Google Calendar API.
Erstelle OAuth 2.0-Anmeldedaten und lade die credentials.json herunter.
3. Erstelle ein Python-Skript für die Automatisierung
Das Skript überwacht die Änderungen in der ICS-Datei und aktualisiert deinen Google Kalender. Der Ablauf ist wie folgt:

Lade die aktuelle ICS-Datei herunter.
Vergleiche sie mit einer vorher gespeicherten Version (falls vorhanden).
Wenn sich die ICS-Datei geändert hat, lade sie neu und aktualisiere den Google Kalender.
Beispielskript:
python
Code kopieren
import os
import requests
import hashlib
from icalendar import Calendar
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from google.oauth2 import service_account

# Pfad zur ICS-Datei und zu den Google Kalender-Anmeldedaten
ICS_URL = "https://dein_stundenplan_url.ics"
ICS_FILE = "stundenplan.ics"
PREVIOUS_HASH_FILE = "previous_hash.txt"

# Google Kalender ID und Anmeldedaten
GOOGLE_CALENDAR_ID = 'dein-google-calendar-id@group.calendar.google.com'
CREDENTIALS_FILE = 'credentials.json'

def download_ics():
    """Lade die ICS-Datei herunter und speichere sie."""
    response = requests.get(ICS_URL)
    with open(ICS_FILE, 'wb') as f:
        f.write(response.content)

def file_hash(file_path):
    """Erstelle einen Hash der Datei."""
    hash_md5 = hashlib.md5()
    with open(file_path, "rb") as f:
        for chunk in iter(lambda: f.read(4096), b""):
            hash_md5.update(chunk)
    return hash_md5.hexdigest()

def ics_has_changed():
    """Überprüfe, ob die ICS-Datei geändert wurde."""
    download_ics()
    
    current_hash = file_hash(ICS_FILE)
    
    if os.path.exists(PREVIOUS_HASH_FILE):
        with open(PREVIOUS_HASH_FILE, 'r') as f:
            previous_hash = f.read()
    else:
        previous_hash = ""

    if current_hash != previous_hash:
        with open(PREVIOUS_HASH_FILE, 'w') as f:
            f.write(current_hash)
        return True
    return False

def update_google_calendar():
    """Aktualisiere den Google Kalender basierend auf der ICS-Datei."""
    creds = None
    if os.path.exists('token.json'):
        creds = Credentials.from_authorized_user_file('token.json', SCOPES)
    else:
        flow = InstalledAppFlow.from_client_secrets_file(CREDENTIALS_FILE, SCOPES)
        creds = flow.run_local_server(port=0)
        with open('token.json', 'w') as token:
            token.write(creds.to_json())

    service = build('calendar', 'v3', credentials=creds)

    # Bestehende Einträge löschen (optional, wenn Du alte Einträge überschreiben willst)
    events_result = service.events().list(calendarId=GOOGLE_CALENDAR_ID).execute()
    for event in events_result.get('items', []):
        service.events().delete(calendarId=GOOGLE_CALENDAR_ID, eventId=event['id']).execute()

    # Neue Einträge aus der ICS-Datei hinzufügen
    with open(ICS_FILE, 'rb') as f:
        cal = Calendar.from_ical(f.read())
        for component in cal.walk():
            if component.name == "VEVENT":
                event = {
                    'summary': str(component.get('summary')),
                    'start': {'dateTime': component.get('dtstart').dt.isoformat()},
                    'end': {'dateTime': component.get('dtend').dt.isoformat()},
                }
                service.events().insert(calendarId=GOOGLE_CALENDAR_ID, body=event).execute()

if ics_has_changed():
    print("ICS-Datei hat sich geändert. Aktualisiere den Google Kalender...")
    update_google_calendar()
else:
    print("Keine Änderungen in der ICS-Datei.")
4. Erläuterung des Skripts
ICS-Download und Überprüfung: Das Skript lädt die ICS-Datei herunter und berechnet den Hash der Datei. Dieser Hash wird mit einem gespeicherten Hash der vorherigen Version der ICS-Datei verglichen. Wenn sich der Hash geändert hat, wird die Datei aktualisiert.
Google Calendar API: Sobald Änderungen erkannt werden, wird die Google Calendar API verwendet, um alte Kalender-Einträge (optional) zu löschen und die neuen aus der aktualisierten ICS-Datei hinzuzufügen.
Google API-Anmeldedaten: Das Skript verwendet OAuth2-Anmeldedaten (credentials.json), um auf den Google Kalender zuzugreifen.
5. Task Scheduler einrichten (für regelmäßige Ausführung)
Damit das Skript regelmäßig ausgeführt wird, kannst du den Windows-Taskplaner verwenden:

Öffne den Taskplaner (taskschd.msc).
Erstelle eine neue Aufgabe.
Stelle die Aufgabe so ein, dass sie täglich oder zu bestimmten Intervallen ausgeführt wird.
Wähle dein Python-Skript als das Programm aus, das ausgeführt werden soll.
6. Überwachung und Anpassungen
Stelle sicher, dass die ICS-URL stabil ist, damit das Skript korrekt arbeiten kann.
Du kannst die Häufigkeit der Überprüfung über den Taskplaner nach deinen Bedürfnissen anpassen.
Mit dieser Methode kannst du sicherstellen, dass dein Google Kalender automatisch aktualisiert wird, sobald Änderungen an deinem Stundenplan auftreten.
