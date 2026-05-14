# 2. DATENMODELLE FÜR A.RED OPERATIONS-SYSTEM

**Ziel:** Definieren aller Kern-Entitäten für die Lead- und Aufgaben-Zentrale, zugeschnitten auf A.RED-spezifische Geschäftslogik.

**Kernprinzip:** FileMaker bleibt Master-System. Diese Modelle beschreiben die MVP-1-Struktur und zeigen gleichzeitig die spätere Zielarchitektur.

**Wichtigste Regel:**
> **Kunde ≠ Standort ≠ Ansprechpartner ≠ Auftrag ≠ Gerät ≠ Rechnungsempfänger**

Diese Trennung ist für A.RED essenziell und muss im Datenmodell sauber sichtbar sein.

---

## 1. MODELLGRENZEN (MVP 1 vs. SPÄTER)

### MVP 1 – vollständig modelliert

- **LEAD** – Eingehende Anfrage, zentrale Erfassung
- **TASK** – Abgeleitete Aufgabe aus beliebiger Quelle
- **CUSTOMER** – Unternehmenskundenmaster
- **SITE** – Standort / Objekt des Kunden
- **CONTACT** – Ansprechpartner
- **SERVICE** – Dienstleistungs-Katalog für A.RED-Leistungen
- **TIME_SLOT** – Zeitfenster-Vorschläge, kein automatisches Booking
- **ASSIGNMENT** – Task-zu-Mitarbeiter-Zuordnung

### Später – als Platzhalter mit grundlegender Struktur

- **OFFER** – Angebot mit Freigabe-Logik
- **ORDER** – Kundenauftrag
- **PURCHASE_ORDER** – Bestellung an Lieferanten
- **DELIVERY** – Wareneingang / Warenübernahme
- **INVOICE** – Rechnung
- **EQUIPMENT** / **FIRE_EXTINGUISHER** – Geräteverwaltung
- **EMPLOYEE** – Mitarbeiterdaten
- **CASHFLOW_ENTRY** – Finanzbewegungen, offene und gebuchte Positionen
- **VEHICLE_STOCK** – Fahrzeuglager

### Nicht Phase 1

- Vollständiger KI-Telefonassistent
- Automatische Terminverschiebung
- Vollständiges Finanzdashboard
- Kundenlogin-Portal
- Fahrzeuglager-Automatisierung
- RZL-Automatisierung
- Automatische Bank- oder Zahlungsaktionen
- Automatische Angebote für komplexe Planleistungen
- GPS-/Geolocation-Überwachung

---

## 2. KERN-ENTITÄTEN

```text
┌─────────────┐         ┌────────┐         ┌──────────────┐
│    LEAD     │────────▶│  TASK  │────────▶│  CUSTOMER    │
└─────────────┘         └────────┘         └──────────────┘
                              │                     │
                              │                     ▼
                              │              ┌──────────────┐
                              │              │    SITE      │
                              │              │ (Standort)   │
                              │              └──────────────┘
                              │                     │
                              ▼                     ▼
                        ┌──────────────┐    ┌──────────────┐
                        │ ASSIGNMENT   │    │   CONTACT    │
                        │(Task→Empl.)  │    │ (Ansprech.)  │
                        └──────────────┘    └──────────────┘
                              │
                              ▼
                        ┌──────────────┐
                        │  TIME_SLOT   │
                        │ (Vorschläge) │
                        └──────────────┘

INDEPENDENT MASTER DATA:
┌──────────────┐    ┌──────────────────────┐
│   SERVICE    │    │ CONTACT_TYPE (enum)  │
│ (A.RED-List) │    │ TASK_PRIORITY (enum) │
└──────────────┘    └──────────────────────┘
```

---

## 3. DETAILLIERTE ENTITY DEFINITIONS

### 3.1 LEAD

**Zweck:** Zentrale Erfassung aller eingehenden Anfragen, unabhängig vom Kanal.

**Wichtig:** Ein Lead ist eine rohe, noch nicht vollständig qualifizierte Anfrage. Nicht alle Leads werden zu Tasks. Ein Lead kann mehrfach verarbeitet werden.

```json
{
  "entity": "LEAD",
  "primaryKey": "lead_id",
  "attributes": {
    "lead_id": {
      "type": "UUID",
      "required": true,
      "description": "Eindeutige Lead-ID, automatisch erzeugt"
    },
    "source_channel": {
      "type": "ENUM",
      "enum": ["phone", "email", "website_form", "manual_entry"],
      "required": true,
      "description": "Erfassungskanal"
    },
    "source_reference": {
      "type": "string",
      "required": false,
      "description": "E-Mail-Message-ID, Telefon-Protokoll-ID, Formular-ID oder sonstige Referenz"
    },
    "received_date": {
      "type": "ISO8601_DateTime",
      "required": true,
      "description": "Zeitstempel der Anfrage"
    },
    "contact_person_name": {
      "type": "string",
      "required": true,
      "maxLength": 255,
      "description": "Name des Anfragenden"
    },
    "contact_phone": {
      "type": "string",
      "required": false,
      "pattern": "^[+\\d\\-() ]{7,25}$",
      "description": "Telefonnummer im internationalen Format"
    },
    "contact_email": {
      "type": "email",
      "required": false,
      "description": "E-Mail-Adresse"
    },
    "company_name": {
      "type": "string",
      "required": false,
      "description": "Firmenname, falls vorhanden"
    },
    "location_description": {
      "type": "string",
      "required": false,
      "description": "Ungefähre Ortsangabe aus der Anfrage"
    },
    "initial_description": {
      "type": "text",
      "required": true,
      "description": "Freitextbeschreibung der Anfrage"
    },
    "urgency_indicator": {
      "type": "ENUM",
      "enum": ["normal", "soon", "urgent", "emergency"],
      "required": false,
      "description": "Zeitliche Dringlichkeit aus der Anfrage"
    },
    "lead_status": {
      "type": "ENUM",
      "enum": ["new", "in_clarification", "classified", "assigned_to_task", "archived", "rejected"],
      "required": true,
      "default": "new"
    },
    "is_existing_customer": {
      "type": "boolean",
      "required": false,
      "description": "Ist der Anfragende bereits Kunde in FileMaker?"
    },
    "linked_customer_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu CUSTOMER, falls bekannt"
    },
    "linked_site_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SITE, falls Standort bekannt"
    },
    "classification_timestamp": {
      "type": "ISO8601_DateTime",
      "required": false
    },
    "classified_service_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SERVICE nach Klassifikation"
    },
    "classification_confidence": {
      "type": "float",
      "min": 0,
      "max": 1,
      "required": false,
      "description": "KI-Confidence für Klassifikation"
    },
    "notes": {
      "type": "text",
      "required": false,
      "description": "Interne Notizen"
    },
    "created_by": {
      "type": "string",
      "required": true,
      "description": "Benutzer oder System, das den Lead erfasst hat"
    },
    "created_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "updated_at": {
      "type": "ISO8601_DateTime",
      "required": true
    }
  },
  "businessRules": [
    "classification_confidence < 0.7 → Lead bleibt auf 'new', KI-Vorschlag nur als Hinweis",
    "Manuelle Genehmigung erforderlich, bevor lead_status auf 'classified' gesetzt wird",
    "Ein Lead kann mehrfach zu verschiedenen Tasks führen",
    "Ein Lead kann verworfen werden, ohne dass ein Task entsteht"
  ]
}
```

---

### 3.2 CUSTOMER

**Zweck:** Kundenmaster mit rechtlichen und kommerziellen Kundendaten.

**Wichtig:** CUSTOMER enthält nicht die Standortadressen. Ein Kunde kann mehrere Standorte haben. Adressen gehören zu SITE, nicht zu CUSTOMER.

```json
{
  "entity": "CUSTOMER",
  "primaryKey": "customer_id",
  "attributes": {
    "customer_id": {
      "type": "UUID",
      "required": true
    },
    "customer_type": {
      "type": "ENUM",
      "enum": ["individual", "company"],
      "required": true
    },
    "status": {
      "type": "ENUM",
      "enum": ["prospect", "active", "inactive", "deactivated", "blocked", "requires_review"],
      "required": true,
      "default": "prospect",
      "description": "Kundenstatus ohne bevorzugte Kundenlogik"
    },
    "name": {
      "type": "string",
      "required": true,
      "maxLength": 255,
      "description": "Firmenname oder vollständiger Name"
    },
    "contact_person": {
      "type": "string",
      "required": false,
      "description": "Hauptansprechpartner, falls bekannt"
    },
    "primary_email": {
      "type": "email",
      "required": false
    },
    "primary_phone": {
      "type": "string",
      "required": false,
      "pattern": "^[+\\d\\-() ]{7,25}$"
    },
    "secondary_phone": {
      "type": "string",
      "required": false
    },
    "website": {
      "type": "url",
      "required": false
    },
    "tax_id": {
      "type": "string",
      "required": false,
      "description": "UID oder Steuernummer"
    },
    "industry_sector": {
      "type": "string",
      "required": false,
      "description": "Branche, z.B. Immobilienverwaltung, Logistik, Einzelhandel"
    },
    "customer_lifetime_value": {
      "type": "float",
      "required": false,
      "description": "Gesamtumsatz EUR aus FileMaker"
    },
    "contract_start_date": {
      "type": "date",
      "required": false
    },
    "last_contact_date": {
      "type": "date",
      "required": false
    },
    "preferred_communication": {
      "type": "ENUM",
      "enum": ["phone", "email", "in_person"],
      "required": false
    },
    "external_id_filemaker": {
      "type": "string",
      "required": false,
      "description": "Kundennummer aus FileMaker, Sync-Key"
    },
    "notes": {
      "type": "text",
      "required": false
    },
    "created_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "updated_at": {
      "type": "ISO8601_DateTime",
      "required": true
    }
  },
  "businessRules": [
    "Bestandskunde bedeutet bessere Zuordnung und FileMaker-Kontext, aber nicht automatisch höhere Priorität",
    "Status 'requires_review' bedeutet: Lead braucht manuellen Check, bevor ein Task entsteht",
    "Status 'blocked' bedeutet: keine neuen Tasks für diesen Kunden anlegen"
  ]
}
```

---

### 3.3 SITE

**Zweck:** Standort oder Objekt eines Kunden. Ein Kunde kann mehrere Sites haben.

**Wichtig:** Adressen, Zugangsnotizen, Prüfintervalle, Standort-Sonderpreise und standortbezogene Rahmenvereinbarungen gehören hier hin, nicht zu CUSTOMER.

```json
{
  "entity": "SITE",
  "primaryKey": "site_id",
  "attributes": {
    "site_id": {
      "type": "UUID",
      "required": true
    },
    "customer_id": {
      "type": "UUID",
      "required": true,
      "description": "FK zu CUSTOMER, muss einem Kunden gehören"
    },
    "site_number_filemaker": {
      "type": "string",
      "required": false,
      "description": "Objektnummer aus FileMaker"
    },
    "external_id_filemaker": {
      "type": "string",
      "required": false,
      "description": "Standort-ID aus FileMaker, Sync-Key"
    },
    "name": {
      "type": "string",
      "required": true,
      "maxLength": 255,
      "description": "Name des Standorts, z.B. 'Wien Zentrale' oder 'Lager Wien'"
    },
    "street_address": {
      "type": "string",
      "required": true,
      "maxLength": 255
    },
    "postal_code": {
      "type": "string",
      "required": true,
      "pattern": "^\\d{4}$",
      "description": "Österreichische Postleitzahl, 4-stellig"
    },
    "city": {
      "type": "string",
      "required": true,
      "maxLength": 100
    },
    "country": {
      "type": "string",
      "required": true,
      "default": "AT",
      "description": "ISO 3166-1 alpha-2, Standard AT"
    },
    "access_notes": {
      "type": "text",
      "required": false,
      "description": "Zugangshinweise, Pförtner, Türöffner, Schlüssel, Ansprechpartner vor Ort"
    },
    "inspection_interval_months": {
      "type": "integer",
      "required": false,
      "description": "Standard-Inspektionsintervall in Monaten"
    },
    "last_inspection_date": {
      "type": "date",
      "required": false
    },
    "next_inspection_due": {
      "type": "date",
      "required": false,
      "description": "Berechnet aus letzter Prüfung und Intervall"
    },
    "special_prices_apply": {
      "type": "boolean",
      "required": false,
      "default": false
    },
    "framework_agreement_id": {
      "type": "string",
      "required": false,
      "description": "Referenz zu einer Rahmenvereinbarung"
    },
    "framework_agreement_number": {
      "type": "string",
      "required": false,
      "description": "Nummer der Rahmenvereinbarung aus FileMaker"
    },
    "notes": {
      "type": "text",
      "required": false
    },
    "is_active": {
      "type": "boolean",
      "required": true,
      "default": true
    },
    "created_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "updated_at": {
      "type": "ISO8601_DateTime",
      "required": true
    }
  },
  "businessRules": [
    "Standort ist nicht gleich Ansprechpartner",
    "Ein Kunde kann 1-n Sites haben",
    "Ein Standort gehört zu genau einem Kunden",
    "Sonderpreise und Rahmenvereinbarungen können später Freigabe-Klassen beeinflussen",
    "Das Inspektionsintervall bestimmt, wann die nächste Inspektion fällig wird"
  ]
}
```

---

### 3.4 CONTACT

**Zweck:** Ansprechpartner bei einem Kunden oder optional bei einem konkreten Standort.

```json
{
  "entity": "CONTACT",
  "primaryKey": "contact_id",
  "attributes": {
    "contact_id": {
      "type": "UUID",
      "required": true
    },
    "customer_id": {
      "type": "UUID",
      "required": true,
      "description": "FK zu CUSTOMER"
    },
    "site_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SITE, falls der Kontakt standortgebunden ist"
    },
    "contact_type": {
      "type": "ENUM",
      "enum": ["person", "department", "role_based"],
      "required": true
    },
    "name": {
      "type": "string",
      "required": true,
      "maxLength": 255
    },
    "title": {
      "type": "string",
      "required": false,
      "description": "Position, z.B. Geschäftsführer oder Objektmanager"
    },
    "email": {
      "type": "email",
      "required": false
    },
    "phone": {
      "type": "string",
      "required": false,
      "pattern": "^[+\\d\\-() ]{7,25}$"
    },
    "is_primary": {
      "type": "boolean",
      "required": true,
      "default": false
    },
    "preferred_communication": {
      "type": "ENUM",
      "enum": ["phone", "email", "sms", "whatsapp"],
      "required": false
    },
    "external_id_filemaker": {
      "type": "string",
      "required": false
    },
    "is_active": {
      "type": "boolean",
      "required": true,
      "default": true
    },
    "created_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "updated_at": {
      "type": "ISO8601_DateTime",
      "required": true
    }
  }
}
```

---

### 3.5 SERVICE

**Zweck:** Katalog der A.RED-Dienstleistungen.

**Wichtig:** Services werden nach Automatisierungs- und Angebotsklasse unterschieden.

```json
{
  "entity": "SERVICE",
  "primaryKey": "service_id",
  "attributes": {
    "service_id": {
      "type": "UUID",
      "required": true
    },
    "name": {
      "type": "string",
      "required": true,
      "maxLength": 255
    },
    "category": {
      "type": "ENUM",
      "enum": [
        "fire_extinguisher_inspection",
        "fire_extinguisher_sales",
        "fire_extinguisher_rental",
        "sorglos_paket_bsb",
        "first_aid_kit",
        "wall_hydrant",
        "rwa_smoke_extraction",
        "fire_protection_plan",
        "escape_route_plan",
        "training",
        "invoice_topic",
        "appointment_change",
        "other",
        "unknown"
      ],
      "required": true,
      "description": "A.RED-spezifische Service-Kategorien"
    },
    "description": {
      "type": "text",
      "required": true
    },
    "offer_class": {
      "type": "ENUM",
      "enum": ["A_automatic", "B_approval", "C_manual_only"],
      "required": true,
      "description": "Freigabe-Klasse für Angebote"
    },
    "automation_level": {
      "type": "ENUM",
      "enum": ["fully_automated", "semi_automated", "manual_only"],
      "required": true
    },
    "typical_duration_minutes": {
      "type": "integer",
      "required": false
    },
    "requires_site_visit": {
      "type": "boolean",
      "required": true,
      "default": false
    },
    "estimated_price_range_min": {
      "type": "float",
      "required": false,
      "description": "EUR"
    },
    "estimated_price_range_max": {
      "type": "float",
      "required": false,
      "description": "EUR"
    },
    "requires_certification": {
      "type": "boolean",
      "required": false
    },
    "external_id_filemaker": {
      "type": "string",
      "required": false
    },
    "keywords_for_matching": {
      "type": "array",
      "items": { "type": "string" },
      "required": false,
      "description": "Begriffe für KI-Klassifikation",
      "example": ["feuerlöscher", "prüfung", "inspektion", "wartung", "tüv"]
    },
    "is_active": {
      "type": "boolean",
      "required": true,
      "default": true
    },
    "created_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "updated_at": {
      "type": "ISO8601_DateTime",
      "required": true
    }
  },
  "businessRules": [
    "Klasse A: automatisch möglich, z.B. Feuerlöscher-Prüfung nach bekannten Regeln",
    "Klasse B: Freigabe nötig, z.B. Sonderpreise, Neukunde, unklares Material",
    "Klasse C: niemals automatisch senden, z.B. Brandschutzpläne oder komplexe Planleistungen",
    "Sorglos-Paket, Feuerlöscher-Prüfung, Feuerlöscher-Verkauf und Feuerlöscher-Miete müssen getrennt bleiben"
  ]
}
```

---

### 3.6 TASK

**Zweck:** Konkrete Aufgabe aus Lead oder anderer Quelle.

**Wichtig:** Tasks sind nicht zwingend an Leads gebunden. Sie können aus E-Mail, Telefon, Website, Angebot, Bestellung, Lieferung, Rechnung, internem Prozess, Lieferant, Kunde oder Steuerberater-Thema entstehen.

```json
{
  "entity": "TASK",
  "primaryKey": "task_id",
  "attributes": {
    "task_id": {
      "type": "UUID",
      "required": true
    },
    "source_type": {
      "type": "ENUM",
      "enum": [
        "lead",
        "email",
        "phone",
        "website",
        "offer",
        "order",
        "purchase_order",
        "delivery",
        "invoice",
        "internal",
        "customer",
        "supplier",
        "tax_advisor",
        "other"
      ],
      "required": true,
      "description": "Quelle der Aufgabe"
    },
    "source_reference": {
      "type": "string",
      "required": false,
      "description": "ID der Quelle, z.B. Lead-ID, E-Mail-ID oder Auftrags-ID"
    },
    "source_lead_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu LEAD, optional falls aus Lead entstanden"
    },
    "task_type": {
      "type": "ENUM",
      "enum": ["inquiry", "follow_up", "scheduling", "offer_prep", "order_handling", "billing", "internal"],
      "required": true
    },
    "service_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SERVICE, falls Service bekannt"
    },
    "customer_id": {
      "type": "UUID",
      "required": true,
      "description": "FK zu CUSTOMER"
    },
    "site_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SITE, falls Standort bekannt"
    },
    "title": {
      "type": "string",
      "required": true,
      "maxLength": 255
    },
    "description": {
      "type": "text",
      "required": true
    },
    "task_class": {
      "type": "ENUM",
      "enum": ["A_automatic", "B_backoffice", "C_management"],
      "required": true,
      "description": "Aufgabenklasse für Routing und Freigabe"
    },
    "priority": {
      "type": "ENUM",
      "enum": ["low", "medium", "high", "critical"],
      "required": true,
      "default": "medium",
      "description": "Berechnet aus Dringlichkeit, Frist, wirtschaftlicher Relevanz und Risiko"
    },
    "status": {
      "type": "ENUM",
      "enum": [
        "created",
        "in_progress",
        "waiting_for_info",
        "ready_to_schedule",
        "scheduled",
        "completed",
        "cancelled",
        "on_hold"
      ],
      "required": true,
      "default": "created"
    },
    "deadline": {
      "type": "ISO8601_DateTime",
      "required": false
    },
    "assigned_to_employee_id": {
      "type": "string",
      "required": false,
      "description": "Mitarbeiter-ID, später FK zu EMPLOYEE"
    },
    "assigned_at": {
      "type": "ISO8601_DateTime",
      "required": false
    },
    "requires_site_visit": {
      "type": "boolean",
      "required": false
    },
    "scheduled_date": {
      "type": "date",
      "required": false
    },
    "scheduled_time_slot_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu TIME_SLOT, falls geplant"
    },
    "is_blocking_existing_appointment": {
      "type": "boolean",
      "required": false,
      "default": false,
      "description": "Kennzeichnet, ob bestehende Kundentermine betroffen wären"
    },
    "notes": {
      "type": "text",
      "required": false
    },
    "external_id_filemaker": {
      "type": "string",
      "required": false
    },
    "created_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "updated_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "completed_at": {
      "type": "ISO8601_DateTime",
      "required": false
    }
  },
  "businessRules": [
    "Klasse A: automatisch oder fast automatisch, z.B. Standardantwort, fehlende Daten anfordern, einfache Angebotsvorbereitung, Nachfass-E-Mail, Erinnerung, Klassifizierung",
    "Klasse B: Büro oder Backoffice, z.B. unklare Anfrage, Sonderpreis, Bestellung, Rechnung prüfen, Materialfrage, Termin mit Risiko, Rahmenvereinbarung prüfen",
    "Klasse C: Geschäftsleitung, z.B. große Freigaben, Preisentscheidung, Rabattentscheidung, problematischer Kunde, Liquiditätsrelevanz, große Bestellung, rechtliches Thema, unwirtschaftlicher Lead, Sonderfall",
    "Priorität ergibt sich aus Dringlichkeit, Frist, wirtschaftlicher Relevanz, Risiko, Aufgabenart, Eskalationsbedarf und Zahlungs- oder Sonderfall",
    "Bestandskunde bedeutet nicht automatisch höhere Priorität",
    "Tasks können aus verschiedenen Quellen entstehen, nicht nur aus Leads"
  ]
}
```

---

### 3.7 TIME_SLOT

**Zweck:** Verfügbare oder gebuchte Zeitfenster. Modell für Terminvorschläge, nicht für automatische Disposition.

**Wichtig:** Das System macht anfangs nur Terminvorschläge. Bestehende Kundentermine dürfen nicht automatisch verschoben werden. Bestätigung durch Frontoffice ist erforderlich.

```json
{
  "entity": "TIME_SLOT",
  "primaryKey": "time_slot_id",
  "attributes": {
    "time_slot_id": {
      "type": "UUID",
      "required": true
    },
    "slot_type": {
      "type": "ENUM",
      "enum": ["available", "booked", "blocked"],
      "required": true,
      "description": "Verfügbar, gebucht oder blockiert"
    },
    "start_datetime": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "end_datetime": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "duration_minutes": {
      "type": "integer",
      "required": true
    },
    "assigned_to_employee_id": {
      "type": "string",
      "required": false,
      "description": "Mitarbeiter-ID"
    },
    "assigned_task_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu TASK, falls gebucht"
    },
    "customer_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu CUSTOMER"
    },
    "site_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SITE, Standort des Termins"
    },
    "location": {
      "type": "string",
      "required": false,
      "description": "Ort aus SITE-Adresse oder manuell"
    },
    "is_changeable": {
      "type": "boolean",
      "required": true,
      "default": false,
      "description": "Kennzeichnet, ob ein Termin noch verschoben oder storniert werden darf"
    },
    "is_proposal_only": {
      "type": "boolean",
      "required": true,
      "default": true,
      "description": "Kennzeichnet einen KI-Vorschlag ohne Kundenbestätigung"
    },
    "proposed_by_ai": {
      "type": "boolean",
      "required": false,
      "description": "Von KI vorgeschlagen"
    },
    "confirmed_by_customer": {
      "type": "boolean",
      "required": false
    },
    "confirmed_at": {
      "type": "ISO8601_DateTime",
      "required": false
    },
    "notes": {
      "type": "text",
      "required": false
    },
    "external_id_google_calendar": {
      "type": "string",
      "required": false,
      "description": "Google Calendar Event ID, schreibender Sync erst nach Frontoffice-Bestätigung"
    },
    "material_risk_identified": {
      "type": "boolean",
      "required": false,
      "description": "Materialrisiko oder Puffer unklar"
    },
    "created_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "updated_at": {
      "type": "ISO8601_DateTime",
      "required": true
    }
  },
  "businessRules": [
    "Google Calendar bleibt vorerst führende Kalenderquelle",
    "System macht nur Terminvorschläge",
    "Bestehende Kundentermine dürfen nicht automatisch verschoben werden",
    "Keine verbindliche Buchung ohne Frontoffice-Bestätigung",
    "Bei Neukunden ist der Materialbedarf oft unklar",
    "Puffer und Anfahrtszeit müssen berücksichtigt werden",
    "Frontoffice entscheidet und bestätigt den Termin"
  ]
}
```

---

### 3.8 ASSIGNMENT

**Zweck:** Zuordnung von Aufgaben zu Mitarbeitern.

```json
{
  "entity": "ASSIGNMENT",
  "primaryKey": ["task_id", "employee_id"],
  "attributes": {
    "task_id": {
      "type": "UUID",
      "required": true
    },
    "employee_id": {
      "type": "string",
      "required": true,
      "description": "Später FK zu EMPLOYEE"
    },
    "assigned_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "assigned_by": {
      "type": "string",
      "required": true
    },
    "role": {
      "type": "ENUM",
      "enum": ["primary_owner", "secondary", "reviewer", "approver"],
      "required": true
    },
    "is_active": {
      "type": "boolean",
      "required": true,
      "default": true
    },
    "unassigned_at": {
      "type": "ISO8601_DateTime",
      "required": false
    },
    "notes": {
      "type": "text",
      "required": false
    }
  }
}
```

---

## 4. SPÄTER – PLATZHALTER FÜR ZIELARCHITEKTUR

### 4.1 OFFER (Phase 2)

```json
{
  "entity": "OFFER",
  "primaryKey": "offer_id",
  "attributes": {
    "offer_id": "UUID",
    "task_id": "UUID, FK zu TASK",
    "customer_id": "UUID",
    "site_id": "UUID",
    "service_ids": "array of UUID",
    "offer_class": "ENUM: A_automatic / B_approval / C_manual_only",
    "status": "ENUM: draft / pending_approval / approved / sent / accepted / rejected",
    "total_amount_eur": "float",
    "approval_chain": "array",
    "approval_status": "pending / approved / rejected",
    "external_id_filemaker": "string",
    "created_at": "ISO8601_DateTime",
    "updated_at": "ISO8601_DateTime"
  }
}
```

### 4.2 ORDER (Phase 2)

```json
{
  "entity": "ORDER",
  "primaryKey": "order_id",
  "attributes": {
    "order_id": "UUID",
    "customer_id": "UUID",
    "site_id": "UUID",
    "offer_id": "UUID, FK zu OFFER",
    "service_ids": "array of UUID",
    "order_number": "string",
    "status": "ENUM: created / scheduled / in_progress / completed / invoiced / cancelled",
    "total_amount_eur": "float",
    "scheduled_date": "date",
    "assigned_employee_id": "string",
    "external_id_filemaker": "string"
  }
}
```

### 4.3 PURCHASE_ORDER (Phase 2)

```json
{
  "entity": "PURCHASE_ORDER",
  "primaryKey": "po_id",
  "attributes": {
    "po_id": "UUID",
    "supplier_id": "UUID",
    "order_date": "date",
    "items": "array of ordered items",
    "total_cost_eur": "float",
    "expected_gross_cash_out_eur": "float",
    "assigned_customer_id": "UUID",
    "assigned_site_id": "UUID",
    "assigned_order_id": "UUID",
    "assigned_employee_id": "string",
    "approval_status": "pending / approved / rejected",
    "status": "ENUM: draft / sent / confirmed / waiting_for_complete_delivery / received / invoiced"
  }
}
```

### 4.4 DELIVERY (Phase 2)

```json
{
  "entity": "DELIVERY",
  "primaryKey": "delivery_id",
  "attributes": {
    "delivery_id": "UUID",
    "purchase_order_id": "UUID",
    "delivery_date": "date",
    "items_received": "array",
    "received_by": "string",
    "is_complete": "boolean",
    "is_damaged": "boolean",
    "photo_documentation": "array of file references",
    "status": "ENUM: pending / incomplete / completed / damaged"
  }
}
```

### 4.5 INVOICE (Phase 2)

```json
{
  "entity": "INVOICE",
  "primaryKey": "invoice_id",
  "attributes": {
    "invoice_id": "UUID",
    "invoice_type": "ENUM: incoming / outgoing",
    "order_id": "UUID",
    "purchase_order_id": "UUID",
    "customer_id": "UUID",
    "supplier_id": "UUID",
    "invoice_number": "string",
    "invoice_date": "date",
    "due_date": "date",
    "net_amount_eur": "float",
    "gross_amount_eur": "float",
    "status": "ENUM: draft / received / checked / approved / exported / paid / overdue",
    "external_id_filemaker": "string",
    "external_id_rzl": "string",
    "created_at": "ISO8601_DateTime"
  }
}
```

### 4.6 EQUIPMENT & FIRE_EXTINGUISHER (Phase 3)

```json
{
  "entity": "EQUIPMENT",
  "primaryKey": "equipment_id",
  "attributes": {
    "equipment_id": "UUID",
    "site_id": "UUID",
    "equipment_type": "ENUM: fire_extinguisher / first_aid_kit / wall_hydrant / other",
    "barcode": "string",
    "serial_number": "string",
    "location_notes": "string",
    "last_inspection_date": "date",
    "next_inspection_due": "date",
    "status": "ENUM: active / expired / maintenance / disposed"
  }
}
```

### 4.7 EMPLOYEE (Phase 2)

```json
{
  "entity": "EMPLOYEE",
  "primaryKey": "employee_id",
  "attributes": {
    "employee_id": "string",
    "name": "string",
    "email": "email",
    "phone": "string",
    "role": "ENUM: frontoffice / backoffice / technician / warehouse / management / admin",
    "is_active": "boolean",
    "home_start_location": "address",
    "skills": "array",
    "monthly_order_budget_eur": "float",
    "monthly_revenue_target_eur": "float"
  }
}
```

### 4.8 CASHFLOW_ENTRY (Phase 2)

```json
{
  "entity": "CASHFLOW_ENTRY",
  "primaryKey": "entry_id",
  "attributes": {
    "entry_id": "UUID",
    "source_type": "ENUM: outgoing_invoice / incoming_invoice / purchase_order / bank_transaction / manual",
    "source_reference": "string",
    "entry_type": "ENUM: expected_in / expected_out / actual_in / actual_out",
    "net_amount_eur": "float",
    "gross_amount_eur": "float",
    "due_date": "date",
    "actual_date": "date",
    "bank_account_id": "string",
    "status": "ENUM: planned / open / booked / overdue / cancelled"
  }
}
```

### 4.9 VEHICLE_STOCK (Phase 3)

```json
{
  "entity": "VEHICLE_STOCK",
  "primaryKey": "vehicle_stock_id",
  "attributes": {
    "vehicle_stock_id": "UUID",
    "vehicle_id": "UUID",
    "license_plate": "string",
    "assigned_employee_id": "string",
    "item_id": "string",
    "quantity": "integer",
    "last_inventory_check": "date",
    "status": "ENUM: active / review_required / inactive"
  }
}
```

---

## 5. INTEGRATIONS-PUNKTE

### FileMaker Sync

FileMaker bleibt das führende System für bestehende Stamm- und Bewegungsdaten.

- `CUSTOMER.external_id_filemaker` ← Kundennummer
- `SITE.external_id_filemaker` ← Standort-ID
- `SERVICE.external_id_filemaker` ← Service-Nummer
- `TASK.external_id_filemaker` ← Aufgabennummer
- `CONTACT.external_id_filemaker` ← Kontakt-ID
- `INVOICE.external_id_filemaker` ← Rechnungsnummer

### Google Workspace

- **Google Calendar → TIME_SLOT** zunächst lesend für Verfügbarkeits- und Terminvorschläge
  - Schreibender Sync erst nach Frontoffice-Bestätigung
  - Keine automatische Verschiebung bestehender Termine
- **Gmail → LEAD** zunächst lesend und klassifizierend
  - E-Mail-Eingang erstellt bei konkretem Bedarf einen Lead mit `source_reference`
  - Bestehende Labels und Postfächer werden schrittweise angebunden

### 3CX / Telefon

- Telefonanlage: 3CX
- Ziel: Transkription, KI-Mitlesen und Antwortvorschläge
- Anfangs keine KI, die selbstständig telefoniert
- Speicherung primär als Transkript für weitere Verarbeitung
- Rechtliche Prüfung vor produktiver Gesprächsaufzeichnung erforderlich

### Datenfluss

```text
Eingang
    ↓
Lead-Erfassung
    ↓
Klassifikation nach SERVICE
    ↓
Task-Erstellung
    ↓
Routing nach task_class A/B/C
    ↓
Mitarbeiter-Zuweisung
    ↓
Terminvorschlag als TIME_SLOT
    ↓
Frontoffice-Bestätigung
    ↓
Google-Calendar-Sync nach Bestätigung
    ↓
Ausführung / Follow-up
    ↓
Später: OFFER / ORDER / INVOICE / CASHFLOW
```

---

## 6. DATENTYPEN & VALIDIERUNG

| Typ | Format | Beispiel | Validierung |
|-----|--------|----------|-------------|
| **UUID** | RFC 4122 | `550e8400-e29b-41d4-a716-446655440000` | 36 Zeichen, Hex + Bindestriche |
| **ISO8601_DateTime** | ISO 8601 | `2026-05-14T19:30:00Z` | UTC-Zeitstempel |
| **date** | ISO 8601 | `2026-05-14` | YYYY-MM-DD |
| **email** | RFC 5322 | `name@example.at` | Standard-E-Mail-Validierung |
| **url** | RFC 3986 | `https://example.at` | Mit Protokoll |
| **postal_code_AT** | Österreich | `1010` | 4-stellig, `^\\d{4}$` |
| **ENUM** | vordefiniert | siehe Entity | Nur definierte Werte |

---

## 7. KRITISCHE GESCHÄFTSREGELN

### 7.1 Kunde ≠ Standort

- **CUSTOMER** enthält Firmenname, UID, Kontaktdaten, Status und Branche.
- **SITE** enthält Adresse, Zugang, Inspektionsintervall und Standortregeln.
- Ein Kunde kann mehrere Sites haben.
- Ein Standort gehört zu genau einem Kunden.

### 7.2 Standort ≠ Ansprechpartner

- **SITE** ist der physische Ort.
- **CONTACT** ist eine Person, Abteilung oder Rolle.
- Ein Kontakt kann optional an eine SITE gebunden sein.
- Ein Standort kann mehrere Ansprechpartner haben.

### 7.3 Lead ≠ Task

- **LEAD** ist die rohe Anfrage.
- **TASK** ist die konkrete, priorisierte Aufgabe.
- Ein Lead kann zu keiner, einer oder mehreren Aufgaben führen.
- Eine Aufgabe kann unabhängig von einem Lead entstehen.

### 7.4 Bestandskunde ≠ höhere Priorität

- Bestandskunde bedeutet bessere Datenqualität und FileMaker-Kontext.
- Bestandskunde bedeutet nicht automatisch höhere Priorität.
- Priorität ergibt sich aus Dringlichkeit, Frist, wirtschaftlicher Relevanz und Risiko.

### 7.5 Komplexe Services bleiben manuell

- Services mit `offer_class = "C_manual_only"` werden nicht automatisch angeboten oder versendet.
- Beispiele: Brandschutzpläne, Fluchtwegpläne, komplexe Planleistungen, haftungsrelevante Sonderfälle.
- KI darf maximal klassifizieren, vorbereiten und benachrichtigen.

### 7.6 Terminmodell = Vorschlag, nicht Disposition

- `is_proposal_only = true` bedeutet KI-Vorschlag, nicht Kundenbestätigung.
- Bestehende Kundentermine dürfen nicht automatisch verschoben werden.
- Verbindliche Buchung erst nach Frontoffice-Bestätigung.
- Google Calendar ist vorerst führende Kalenderquelle.
- Schreibender Google-Calendar-Sync erfolgt erst nach Frontoffice-Bestätigung.

### 7.7 Angebotsklassen

- **Klasse A – automatisch möglich:** Standardprodukte, klare Stückzahlen, bekannte Regeln, Sorglos-Paket, Feuerlöscher nach Kalkulator.
- **Klasse B – Freigabe nötig:** Sonderpreise, Neukunde, unklare Daten, Rahmenvereinbarung, Zahlungsproblem, Materialrisiko.
- **Klasse C – niemals automatisch senden:** Planleistungen, haftungsrelevante Leistungen, Sonderprojekte, komplexe Erstberatungen.

---

## 8. BEISPIEL-DATENSÄTZE

### Beispiel 1: Neuer Lead aus Telefon

```json
{
  "lead_id": "550e8400-e29b-41d4-a716-446655440001",
  "source_channel": "phone",
  "received_date": "2026-05-14T10:30:00Z",
  "contact_person_name": "Eva Pokorny",
  "contact_phone": "+43 1 234567",
  "company_name": "Pokorny Hausverwaltung GmbH",
  "location_description": "Wien 3. Bezirk",
  "initial_description": "Benötigen Feuerlöscher-Inspektionen für 5 verwaltete Objekte",
  "urgency_indicator": "normal",
  "lead_status": "new",
  "is_existing_customer": true,
  "linked_customer_id": "550e8400-e29b-41d4-a716-446655440100",
  "created_by": "frontoffice",
  "created_at": "2026-05-14T10:35:00Z"
}
```

### Beispiel 2: SITE mit Inspektionsintervall

```json
{
  "site_id": "550e8400-e29b-41d4-a716-446655440201",
  "customer_id": "550e8400-e29b-41d4-a716-446655440100",
  "name": "Pokorny Objekt 1 – Wien Höchstädtplatz",
  "street_address": "Höchstädtplatz 9",
  "postal_code": "1030",
  "city": "Wien",
  "country": "AT",
  "site_number_filemaker": "OBJ-2024-001",
  "access_notes": "Pförtner im Büro, Türöffner bei Maria Doppler",
  "inspection_interval_months": 12,
  "last_inspection_date": "2025-04-15",
  "next_inspection_due": "2026-04-15",
  "special_prices_apply": false,
  "framework_agreement_number": "RA-HV-2024-001"
}
```

### Beispiel 3: Task aus Lead

```json
{
  "task_id": "550e8400-e29b-41d4-a716-446655440003",
  "source_type": "lead",
  "source_reference": "550e8400-e29b-41d4-a716-446655440001",
  "source_lead_id": "550e8400-e29b-41d4-a716-446655440001",
  "task_type": "scheduling",
  "service_id": "550e8400-e29b-41d4-a716-446655440002",
  "customer_id": "550e8400-e29b-41d4-a716-446655440100",
  "title": "Feuerlöscher-Inspektionen – 5 Objekte Pokorny",
  "description": "Inspektionen für 5 verwaltete Objekte fällig. Rahmenvereinbarung prüfen.",
  "task_class": "B_backoffice",
  "priority": "medium",
  "status": "created",
  "assigned_to_employee_id": null,
  "created_at": "2026-05-14T10:40:00Z"
}
```

---

## 9. MVP-1-ZUSAMMENFASSUNG

### Vollständig modelliert

- LEAD
- TASK
- CUSTOMER
- SITE
- CONTACT
- SERVICE
- TIME_SLOT
- ASSIGNMENT

### Funktionalität MVP 1

- Zentrale Lead-Erfassung über Telefon, E-Mail, Website und manuelle Eingabe
- Lead-Klassifikation mit KI und Confidence-Check
- Task-Routing nach Klasse A/B/C
- Task-Zuweisung an Mitarbeiter oder Rolle
- Terminvorschläge auf Basis von Kalenderdaten
- Bestandskunden-Datenabgleich mit FileMaker
- Standort-Management mit Inspektionsintervallen, Sonderpreisen und Rahmenvereinbarungen

### Nicht MVP 1

- Automatische Angebote
- Automatische Bestellungen
- Rechnungsautomatisierung
- Geräte- und Ausrüstungsmanagement
- Fahrzeuglager
- Finanzdashboard
- Produktiver Telefonassistent
- GPS-/Geolocation-Überwachung

---

## 10. OFFENE FRAGEN ZUR KLÄRUNG

1. **Rahmenvereinbarungen:** Separate Entität oder vorerst Referenzfeld an CUSTOMER/SITE ausreichend?

2. **Sonderpreise:** Standortbezogen, kundenbezogen oder beides?

3. **Inspektionsintervalle:** Universal pro Service-Typ oder pro Standort variabel?

4. **Google Calendar:** Dauerhaft führende Kalenderquelle oder später Ablöse durch eigenes Scheduling-Modul?

5. **E-Mail-Integration:** Welche Postfächer und Labels werden zuerst produktiv angebunden?

6. **Telefon-Klassifikation:** Technische Umsetzung für 3CX-Aufzeichnung, Transkription, Speicherung und rechtliche Freigabe klären.

7. **Geschäftsleitung-Approval:** Wer sind die konkreten C-Approver für `task_class = "C_management"`?

8. **Material-Risiko:** Welche Regeln erkennt die KI für `TIME_SLOT.material_risk_identified`?

9. **Terminbestätigung:** Über welchen Kanal erfolgt die Bestätigung: E-Mail, Telefon, SMS oder Portal?

10. **Task-Benachrichtigung:** Wer wird über neu erstellte Tasks welcher Klasse benachrichtigt?

---

## 11. NEXT STEPS

1. Datenmodelle als Grundlage verwenden
2. JSON-Schemas für API-Validierung erstellen: `/schemas/`
3. Geschäftsregeln dokumentieren: `docs/3_GESCHAEFTSREGELN.md`
4. Integrationsplan dokumentieren: `docs/4_INTEGRATIONSPLAN.md`
5. Entscheidungs-Log anlegen: `decisions/ADR-001.md`, `decisions/ADR-002.md`, ...

---

## 12. QUALITÄTSCHECK

Nach Änderungen an dieser Datei muss geprüft werden, dass folgende Begriffe nicht vorkommen:

```text
vip
VIP
München
Geschätslogik
Kundengehören
Datenflusss
bidirektional
internationale Format
```

Folgende Begriffe sollen vorkommen:

```text
Geschäftslogik
Wien Zentrale
muss einem Kunden gehören
Datenfluss
schreibender Sync erst nach Frontoffice-Bestätigung
Keine automatische Verschiebung bestehender Termine
```

---

**Version:** 2.1  
**Status:** ÜBERARBEITET UND BEREINIGT  
**Letzte Änderung:** 2026-05-14  
**Österreichische Standards:** AT-defaults, 4-stellige PLZ  
**A.RED-spezifisch:** Services, Site-Modell, keine bevorzugte Kundenlogik  
**FileMaker Master:** Sync-Keys vorgesehen  
**MVP-1-Grenzen:** Klar dokumentiert
