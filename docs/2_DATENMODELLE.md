# 2. DATENMODELLE FÜR A.RED OPERATIONS-SYSTEM

**Ziel:** Definieren aller Kern-Entitäten für die Lead- und Aufgaben-Zentrale, zugeschnitten auf A.RED-spezifische Geschätslogik.

**Kernprinzip:** FileMaker bleibt Master-System. Diese Modelle beschreiben die MVP-1-Struktur und zeigen gleichzeitig die spätere Zielarchitektur.

**Wichtigste Regel:**
> **Kunde ≠ Standort ≠ Ansprechpartner ≠ Auftrag ≠ Gerät ≠ Rechnungsempfänger**

Diese Trennung ist für A.RED essentiell und muss im Datenmodell sauber sichtbar sein.

---

## 1. MODELLGRENZEN (MVP 1 vs. SPÄTER)

### MVP 1 – Vollständig modelliert
- **LEAD** – Eingehende Anfrage, zentrale Erfassung
- **TASK** – Abgeleitete Aufgabe aus beliebiger Quelle
- **CUSTOMER** – Unternehmenskundenmaster
- **SITE** – ⭐ NEU: Standort / Objekt des Kunden
- **CONTACT** – Ansprechpartner
- **SERVICE** – Dienstleistungs-Katalog (A.RED-Leistungen)
- **TIME_SLOT** – Zeitfenster-Vorschläge (kein automatisches Booking)
- **ASSIGNMENT** – Task → Mitarbeiter-Zuordnung

### Später – Als Platzhalter mit grundlegender Struktur
- **OFFER** – Angebot (mit Freigabe-Logik)
- **ORDER** – Kundenauftrag
- **PURCHASE_ORDER** – Bestellungen an Lieferanten
- **DELIVERY** – Wareneingang/Versand
- **INVOICE** – Rechnung/Billing
- **EQUIPMENT** / **FIRE_EXTINGUISHER** – Geräteverwaltung
- **EMPLOYEE** – Mitarbeiterdaten
- **CASHFLOW_ENTRY** – Finanzbewegungen (Booked, Open)
- **VEHICLE_STOCK** – Fahrzeuglager

**Nicht Phase 1:**
- ❌ Vollständiger KI-Telefonassistent
- ❌ Automatische Terminverschiebung
- ❌ Finanzdashboard (vollständig)
- ❌ Kundenlogin-Portal
- ❌ Fahrzeuglager-Automatisierung
- ❌ RZL-Automatisierung
- ❌ Automatische Bank-/Zahlungsaktionen
- ❌ Automatische Angebote für komplexe Planleistungen
- ❌ GPS/Geolocation-Überwachung

---

## 2. KERN-ENTITÄTEN (ENTITY RELATIONSHIP DIAGRAM)

```
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

**Wichtig:** Ein Lead ist eine **rohe, unklassifizierte Anfrage**. Nicht alle Leads werden zu Tasks. Ein Lead kann mehrfach verarbeitet werden.

```json
{
  "entity": "LEAD",
  "primaryKey": "lead_id",
  "attributes": {
    "lead_id": {
      "type": "UUID",
      "required": true,
      "description": "Eindeutige Lead-ID (auto-generated)"
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
      "description": "Email-Message-ID, Telefon-Protokoll-ID, Form-Submission-ID, etc."
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
      "description": "Telefonnummer (internationale Format)"
    },
    "contact_email": {
      "type": "email",
      "required": false,
      "description": "E-Mail-Adresse"
    },
    "company_name": {
      "type": "string",
      "required": false,
      "description": "Firmenname (falls vorhanden)"
    },
    "location_description": {
      "type": "string",
      "required": false,
      "description": "Ungefähre Ortsangabe aus Anfrage"
    },
    "initial_description": {
      "type": "text",
      "required": true,
      "description": "Freitextbeschreibung der Anfrage (wie eingegeben)"
    },
    "urgency_indicator": {
      "type": "ENUM",
      "enum": ["normal", "soon", "urgent", "emergency"],
      "required": false,
      "description": "Zeitliche Dringlichkeit (aus Anfrage erkannt)"
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
      "description": "Ist bereits Kunde in FileMaker?"
    },
    "linked_customer_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu CUSTOMER (falls bekannt oder Bestandskunde)"
    },
    "linked_site_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SITE (falls Standort bekannt)"
    },
    "classification_timestamp": {
      "type": "ISO8601_DateTime",
      "required": false
    },
    "classified_service_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SERVICE (nach Klassifikation)"
    },
    "classification_confidence": {
      "type": "float",
      "min": 0,
      "max": 1,
      "required": false,
      "description": "KI-Confidence für Klassifikation (0.0–1.0)"
    },
    "notes": {
      "type": "text",
      "required": false,
      "description": "Interne Notizen"
    },
    "created_by": {
      "type": "string",
      "required": true,
      "description": "Benutzer, der Lead erfasst hat"
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
    "classification_confidence < 0.7 → Lead bleibt auf 'new', KI-Vorschlag nur als Notification",
    "Manuelle Genehmigung erforderlich, bevor lead_status → 'classified'",
    "Ein Lead kann mehrfach zu verschiedenen Tasks führen",
    "Ein Lead kann verworfen werden ohne Task"
  ]
}
```

---

### 3.2 CUSTOMER
**Zweck:** Kundenmaster – enthält rechtliche/kommerzielle Kundendaten. **NICHT die Adresse(n)!**

**Wichtig:** Ein Kunde kann mehrere Standorte (SITE) haben. Adressen gehören zu SITE, nicht zu CUSTOMER.

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
      "description": "Ansprechpartner (bei company_type=company)"
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
      "description": "UID (Österreich) oder Steuernummer"
    },
    "industry_sector": {
      "type": "string",
      "required": false,
      "description": "Branche (z.B. 'Immobilienverwaltung', 'Logistik', 'Einzelhandel')"
    },
    "customer_lifetime_value": {
      "type": "float",
      "required": false,
      "description": "Gesamtumsatz EUR (aus FileMaker)"
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
      "description": "Kundennummer aus FileMaker (Sync-Key)"
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
    "Bestandskunde (status='active') bedeutet: bessere Zuordnung, FileMaker-Daten prüfen, aber NICHT automatisch höhere Priorität",
    "Status 'requires_review' = Lead braucht manuellen Check bevor Task entsteht",
    "Status 'blocked' = keine neuen Tasks für diesen Kunden anlegen"
  ]
}
```

---

### 3.3 SITE (⭐ NEU)
**Zweck:** Standort / Objekt eines Kunden. Ein Kunde kann mehrere Sites haben.

**KRITISCH:** Alle Adressen, Zugangsnotizen, Prüfintervalle, Sonderpreise gehören hier hin, NICHT zu CUSTOMER.

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
      "description": "FK zu CUSTOMER (Muss ein Kundengehören)"
    },
    "site_number_filemaker": {
      "type": "string",
      "required": false,
      "description": "Objektnummer aus FileMaker"
    },
    "external_id_filemaker": {
      "type": "string",
      "required": false,
      "description": "Standort-ID aus FileMaker (Sync-Key)"
    },
    "name": {
      "type": "string",
      "required": true,
      "maxLength": 255,
      "description": "Name des Standorts (z.B. 'München Zentrale', 'Lager Wien')"
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
      "description": "Österreich: 4-stellig"
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
      "description": "ISO 3166-1 alpha-2 (default: AT)"
    },
    "access_notes": {
      "type": "text",
      "required": false,
      "description": "Zugangshinweise, Pförtner, Türöffner, etc."
    },
    "inspection_interval_months": {
      "type": "integer",
      "required": false,
      "description": "Standard-Inspektionsintervall (z.B. 12)"
    },
    "last_inspection_date": {
      "type": "date",
      "required": false
    },
    "next_inspection_due": {
      "type": "date",
      "required": false,
      "description": "Berechnet: last_inspection_date + interval_months"
    },
    "special_prices_apply": {
      "type": "boolean",
      "required": false,
      "default": false
    },
    "framework_agreement_id": {
      "type": "string",
      "required": false,
      "description": "Referenz zu Rahmenvereinbarung (external)"
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
    "Standort ist nicht gleich Ansprechpartner (Contact ist separate Entität)",
    "Special prices und Rahmenvereinbarungen ermöglichen später Freigabe-Klassen A/B/C",
    "Inspektionsintervall bestimmt, wann nächste Inspektion fällig wird",
    "Ein Kunden kann 1-n Sites haben (z.B. Hausverwaltung mit 20 Objekten)"
  ]
}
```

---

### 3.4 CONTACT
**Zweck:** Ansprechpartner bei einem Kunden. Ein Kunde kann mehrere Ansprechpartner haben.

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
      "description": "FK zu SITE (optional, falls Contact an Standort gebunden)"
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
      "description": "Position (z.B. 'Geschäftsführer', 'Objektmanager')"
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
**Zweck:** Katalog der A.RED-Dienstleistungen (statische Master-Daten).

**Wichtig:** Services werden nach Automatisierungs-Klasse unterschieden (siehe Punkt 6).

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
      "description": "Freigabe-Klasse für Angebote (siehe Punkt 6)"
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
      "description": "Für KI-Klassifikation",
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
    "Klasse A: automatisch möglich (z.B. Feuerlöscher-Prüfung nach bekanntem Plan)",
    "Klasse B: Freigabe nötig (z.B. Sonderpreise, Neukunde, unklares Material)",
    "Klasse C: niemals automatisch (z.B. Brandschutzpläne, Planleistungen)",
    "Sorglos-Paket, Feuerlöscher-Prüfung und -Verkauf müssen getrennt sein"
  ]
}
```

---

### 3.6 TASK
**Zweck:** Konkrete Aufgabe (kann aus Lead oder anderen Quellen entstehen).

**KRITISCH:** Tasks sind nicht zwingend an LEAD gebunden. Sie entstehen aus verschiedenen Quellen.

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
      "description": "Woher kommt diese Aufgabe?"
    },
    "source_reference": {
      "type": "string",
      "required": false,
      "description": "ID der Quelle (Lead-ID, Email-ID, Order-ID, etc.)"
    },
    "source_lead_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu LEAD (optional, falls aus Lead entstanden)"
    },
    "task_type": {
      "type": "ENUM",
      "enum": ["inquiry", "follow_up", "scheduling", "offer_prep", "order_handling", "billing", "internal"],
      "required": true
    },
    "service_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SERVICE (optional, falls Service bekannt)"
    },
    "customer_id": {
      "type": "UUID",
      "required": true,
      "description": "FK zu CUSTOMER"
    },
    "site_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SITE (falls Standort bekannt)"
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
      "description": "Aufgabenklasse für Routing"
    },
    "priority": {
      "type": "ENUM",
      "enum": ["low", "medium", "high", "critical"],
      "required": true,
      "default": "medium",
      "description": "Wird aus Urgenz, Frist, wirtschaftlicher Relevanz, Risiko berechnet"
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
      "description": "Mitarbeiter-ID (später FK zu EMPLOYEE)"
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
      "description": "FK zu TIME_SLOT (falls geplant)"
    },
    "is_blocking_existing_appointment": {
      "type": "boolean",
      "required": false,
      "default": false,
      "description": "WICHTIG: Verschiebt das einen bestehenden Kundentermin?"
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
    "Klasse A: automatisch/fast automatisch (Standardantwort, fehlende Daten anfordern, einfache Angebotsvorbereitung, Nachfass-Email, Erinnerung, Klassifizierung)",
    "Klasse B: Büro/Backoffice (unklare Anfrage, Sonderpreis, Bestellung, Rechnung prüfen, Materialfrage, Termin mit Risiko, Rahmenvereinbarung prüfen)",
    "Klasse C: Geschäftsleitung (große Freigaben, Preis-/Rabattentscheidung, problematischer Kunde, Liquiditätsrelevanz, große Bestellung, rechtliches, unwirtschaftlicher Lead, Sonderfall)",
    "Priorität ergibt sich aus: Dringlichkeit, Frist, wirtschaftlicher Relevanz, Risiko, Aufgabenart, Eskalationsbedarf, Zahlungs-/Sonderfall",
    "Bestandskunde ist nicht gleich höhere Priorität",
    "Tasks können aus verschiedenen Quellen entstehen, nicht nur aus Leads"
  ]
}
```

---

### 3.7 TIME_SLOT
**Zweck:** Verfügbare/gebuchte Zeitfenster. Modell für Terminvorschläge, nicht für automatische Disposition.

**WICHTIG:** Das System macht anfangs nur Terminvorschläge. Bestehende Kundentermine dürfen nicht automatisch verschoben werden. Bestätigung durch Frontoffice ist erforderlich.

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
      "description": "Verfügbar / Gebucht / Blockiert (z.B. Urlaub)"
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
      "description": "FK zu TASK (falls booked)"
    },
    "customer_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu CUSTOMER"
    },
    "site_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu SITE (Standort des Termins)"
    },
    "location": {
      "type": "string",
      "required": false,
      "description": "Ort (aus SITE.address oder manuell)"
    },
    "is_changeable": {
      "type": "boolean",
      "required": true,
      "default": false,
      "description": "Kann Termin noch verschoben/storniert werden?"
    },
    "is_proposal_only": {
      "type": "boolean",
      "required": true,
      "default": false,
      "description": "KI-Vorschlag oder bereits Kundenbestätigung?"
    },
    "proposed_by_ai": {
      "type": "boolean",
      "required": false,
      "description": "Von KI vorgeschlagen?"
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
      "description": "Google Calendar Event ID (bidirektionaler Sync)"
    },
    "material_risk_identified": {
      "type": "boolean",
      "required": false,
      "description": "Materialrisiko oder Puffer unklar?"
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
    "Google Kalender bleibt vorerst führende Quelle",
    "System macht nur Terminvorschläge (is_proposal_only=true)",
    "Bestehende Kundentermine: is_changeable=false → keine automatische Verschiebung",
    "Keine verbindliche Buchung ohne Frontoffice-Bestätigung",
    "Bei Neukunden: Materialbedarf oft unklar (material_risk_identified=true)",
    "Puffer und Anfahrtszeit müssen berücksichtigt werden",
    "Frontoffice entscheidet und bestätigt Termin"
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
**Minimal Structure für späteren Ausbau:**
```json
{
  "entity": "OFFER",
  "primaryKey": "offer_id",
  "attributes": {
    "offer_id": "UUID",
    "task_id": "UUID (FK zu TASK)",
    "customer_id": "UUID",
    "site_id": "UUID",
    "service_ids": "array of UUID",
    "offer_class": "ENUM (A_automatic / B_approval / C_manual_only)",
    "status": "ENUM (draft / pending_approval / approved / sent / accepted / rejected)",
    "total_amount_eur": "float",
    "approval_chain": "array (wer muss freigeben?)",
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
    "offer_id": "UUID (FK)",
    "service_ids": "array of UUID",
    "order_number": "string",
    "status": "ENUM (created / scheduled / in_progress / completed / invoiced / cancelled)",
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
    "items": "array (supplier items)",
    "total_cost_eur": "float",
    "status": "ENUM (draft / sent / confirmed / partial / received / invoiced)"
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
    "order_id": "UUID",
    "delivery_date": "date",
    "items_received": "array",
    "received_by": "string",
    "status": "ENUM (pending / partial / completed / damaged)"
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
    "order_id": "UUID (FK)",
    "customer_id": "UUID",
    "invoice_number": "string",
    "invoice_date": "date",
    "due_date": "date",
    "total_amount_eur": "float",
    "status": "ENUM (draft / sent / opened / overdue / paid / written_off)",
    "external_id_filemaker": "string",
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
    "equipment_type": "ENUM (fire_extinguisher / first_aid_kit / wall_hydrant / etc.)",
    "serial_number": "string",
    "location_notes": "string",
    "last_inspection_date": "date",
    "next_inspection_due": "date",
    "status": "ENUM (active / expired / maintenance / disposed)"
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
    "role": "ENUM (sales / technician / manager / admin)",
    "is_active": "boolean",
    "skills": "array (z.B. ['Feuerlöscher-Prüfung', 'Ersthelfer'])"
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
    "invoice_id": "UUID",
    "entry_type": "ENUM (booked / open / overdue)",
    "amount_eur": "float",
    "due_date": "date",
    "received_date": "date"
  }
}
```

### 4.9 VEHICLE_STOCK (Phase 3)
```json
{
  "entity": "VEHICLE_STOCK",
  "primaryKey": "vehicle_id",
  "attributes": {
    "vehicle_id": "UUID",
    "license_plate": "string",
    "capacity": "object (Feuerlöscher-Kapazität, Platz für Ausrüstung, etc.)",
    "assigned_employee_id": "string",
    "status": "ENUM (available / in_use / maintenance / decommissioned)"
  }
}
```

---

## 5. INTEGRATIONS-PUNKTE

### FileMaker Sync (Master)
- `CUSTOMER.external_id_filemaker` ← Kundennummer
- `SITE.external_id_filemaker` ← Standort-ID
- `SERVICE.external_id_filemaker` ← Service-Nummer
- `TASK.external_id_filemaker` ← Aufgabennummer
- `CONTACT.external_id_filemaker` ← Kontakt-ID
- `INVOICE.external_id_filemaker` ← Rechnungsnummer

### Google Workspace
- **Google Calendar ↔ TIME_SLOT** (bidirektional)
  - Neue gebuchte TIME_SLOT erzeugen Google Calendar Event
  - Google Calendar Änderungen aktualisieren TIME_SLOT
- **Gmail ↔ LEAD** (unidirektional, vorerst)
  - Email-Eingang erstellt Lead mit source_reference

### Datenflusss
```
Eingang (Lead)
    ↓
Klassifikation (SERVICE)
    ↓
Task-Erstellung (TASK)
    ↓
Routing (task_class A/B/C)
    ↓
Mitarbeiter-Zuweisung (ASSIGNMENT)
    ↓
Terminvorschlag (TIME_SLOT, is_proposal_only)
    ↓
Frontoffice-Bestätigung
    ↓
Google Calendar Sync (external_id_google_calendar)
    ↓
Ausführung / Follow-up
    ↓
→ später: OFFER / ORDER / INVOICE
```

---

## 6. DATENTYPEN & VALIDIERUNG

| Typ | Format | Beispiel | Validierung |
|-----|--------|---------|------------|
| **UUID** | RFC 4122 | `550e8400-e29b-41d4-a716-446655440000` | 36 chars, hex+hyphens |
| **ISO8601_DateTime** | ISO 8601 | `2026-05-14T19:30:00Z` | UTC timezone |
| **date** | ISO 8601 | `2026-05-14` | YYYY-MM-DD |
| **email** | RFC 5322 | `name@example.com` | Standard-Validierung |
| **url** | RFC 3986 | `https://example.at` | Mit Protokoll |
| **postal_code_AT** | Österreich | `1010` | 4-stellig, ^\d{4}$ |
| **ENUM** | Predefined | siehe Entity | Nur vordefinierte Werte |

---

## 7. KRITISCHE GESCHÄFTSREGELN

### 7.1 Kunde ≠ Standort
- **CUSTOMER** enthält: Firmenname, UID, Kontaktdaten, Status, Branche
- **SITE** enthält: Adresse, Zugang, Inspektionsintervall, Rahmenvertrag
- Ein Kunde kann 1-n Sites haben
- Ein Standort gehört zu genau einem Kunden

### 7.2 Standort ≠ Ansprechpartner
- **SITE** = physischer Ort
- **CONTACT** = Person bei CUSTOMER (optional gebunden an SITE)
- Ein Kontakt kann mehrere Sites verwalten
- Ein Standort kann mehrere Ansprechpartner haben

### 7.3 Lead ≠ Task
- **LEAD** = rohe, unklassifizierte Anfrage
- **TASK** = konkrete, priorisierte Aufgabe
- Ein Lead kann zu 0 oder mehreren Tasks führen
- Eine Task kann unabhängig vom Lead entstehen

### 7.4 Bestandskunde ≠ höhere Priorität
- Bestandskunde bedeutet: bessere Datenqualität, FileMaker-Kontext vorhanden
- **Aber nicht automatisch höhere Priorität**
- Priorität ergibt sich aus: Dringlichkeit, Frist, wirtschaftlicher Relevanz, Risiko

### 7.5 Komplexe Services = immer manuell
- Services mit `offer_class="C_manual_only"` werden NIEMALS automatisch verarbeitet
- Beispiele: Brandschutzpläne, Planleistungen, haftungsrelevante Leistungen
- KI darf maximal klassifizieren und benachrichtigen

### 7.6 Terminmodell = Vorschlag, nicht Disposition
- `is_proposal_only=true` = KI-Vorschlag, nicht Kundenbestätigung
- Bestehende Kundentermine: `is_changeable=false` (keine automatische Verschiebung)
- Nur mit Frontoffice-Bestätigung → `confirmed_by_customer=true`
- Google Calendar ist Master-Quelle (vorerst)

### 7.7 Angebot-Klassen (später Automatisierung)
- **Klasse A (automatisch):** Standardprodukte, bekannte Bestandskunden, klare Stückzahlen
- **Klasse B (Freigabe nötig):** Sonderpreise, Neukunde, unklare Daten, Rahmenvereinbarung, Zahlungsproblem
- **Klasse C (niemals automatisch):** Planleistungen, haftungsrelevant, komplex

---

## 8. BEISPIEL-DATENSÄTZE (ÖSTERREICH)

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
  "created_by": "anna.mueller",
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

### Beispiel 3: Task aus Lead (Klasse B)
```json
{
  "task_id": "a716-446655440003-550e8400-e29b-41d4",
  "source_type": "lead",
  "source_reference": "550e8400-e29b-41d4-a716-446655440001",
  "source_lead_id": "550e8400-e29b-41d4-a716-446655440001",
  "task_type": "scheduling",
  "service_id": "e29b-41d4-550e-8400-a716446655440002",
  "customer_id": "550e8400-e29b-41d4-a716-446655440100",
  "title": "Feuerlöscher-Inspektionen – 5 Objekte Pokorny",
  "description": "Inspektionen für 5 verwaltete Objekte fällig (Rahmenvertrag RA-HV-2024-001)",
  "task_class": "B_backoffice",
  "priority": "medium",
  "status": "created",
  "assigned_to_employee_id": null,
  "created_at": "2026-05-14T10:40:00Z"
}
```

---

## 9. MVP-1-ZUSAMMENFASSUNG

### Vollständig modelliert & umgesetzt
✅ LEAD  
✅ TASK  
✅ CUSTOMER  
✅ SITE (⭐ NEU)  
✅ CONTACT  
✅ SERVICE  
✅ TIME_SLOT (Vorschlagsmodell)  
✅ ASSIGNMENT  

### Funktionalität MVP 1
- Zentrale Lead-Erfassung (4 Kanäle)
- Lead-Klassifikation (KI mit Confidence-Check)
- Task-Routing nach Klasse (A/B/C)
- Task-Zuweisung an Mitarbeiter
- Terminvorschläge (Google Calendar lesbar)
- Bestandskunden-Datenabgleich (FileMaker)
- Standort-Management (Inspektionsintervalle, Sonderpreise)

### Nicht MVP 1
- Automatische Angebote
- Automatische Bestellungen
- Rechnungsautomatisierung
- Geräte-/Ausrüstungs-Management
- Fahrzeuglager
- Finanzdashboard
- Telefonassistent (produktiv)
- GPS/Geolocation

---

## 10. OFFENE FRAGEN ZUR KLÄRUNG

1. **Rahmenvereinbarungen:** Sollten als separate Entität modelliert werden oder reicht `SITE.framework_agreement_id`?

2. **Sonderpreise:** Sind diese Site-spezifisch oder auch kunden-spezifisch? (aktuell: Site-spezifisch)

3. **Inspektionsintervalle:** Sind diese universal pro Service-Typ (z.B. alle Feuerlöscher 12 Monate) oder Site-spezifisch variabel?

4. **Google Calendar:** Bleibt das dauerhaft die Master-Quelle oder wird das später durch eigenes Scheduling ersetzt?

5. **Email-Integration:** Sollen Emails automatisch als LEAD erfasst werden oder nur manuell?

6. **Telefon-Klassifikation:** Gibt es ein Telefonsystem, das Anrufe aufzeichnet/transkribiert?

7. **Geschäftsleitung-Approval:** Wer sind die C-Approver für `task_class="C_management"`?

8. **Material-Risiko:** Wie wird `TIME_SLOT.material_risk_identified` von der KI erkannt?

9. **Bestätigung Kundentermin:** Über welchen Kanal erfolgt die Terminbestätigung? (Email, SMS, Call, Portal?)

10. **Task-Benachrichtigung:** Wer wird über neu erstellte Tasks welcher Klasse benachrichtigt?

---

## 11. NEXT STEPS

1. ✅ **Datenmodelle überarbeitet** (diese Datei)
2. 📋 **→ JSON-Schemas** für API-Validierung (`/schemas/`)
3. 📋 **→ Geschäftsregeln** (doc/3_GESCHAEFTREGELN.md)
4. 📋 **→ Integration-Plan** (doc/4_INTEGRATIONS-PLAN.md)
5. 📋 **→ Entscheidungs-Log** (decisions/ADR-001, ADR-002, ...)

---

**Version:** 2.0  
**Status:** ÜBERARBEITET (auf Basis detaillierter A.RED-Anforderungen)  
**Letzte Änderung:** 2026-05-14  
**Österreichische Standards:** ✅ AT-defaults, 4-stellige PLZ  
**A.RED-spezifisch:** ✅ Services, Site-Modell, keine bevorzugte Kundenlogik
**FileMaker Master:** ✅ Sync-Keys vorgesehen  
**MVP-1-Grenzen:** ✅ Klar dokumentiert  
