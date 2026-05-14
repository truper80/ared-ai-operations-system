# 2. DATENMODELLE FÜR A.RED OPERATIONS-SYSTEM

**Ziel:** Definieren aller Kern-Entitäten, Beziehungen und Attribute für die Lead- und Aufgaben-Zentrale.

**Prinzip:** FileMaker bleibt Master-System. Diese Modelle beschreiben, wie Daten strukturiert werden, bevor sie in FileMaker oder Google Workspace fließen.

---

## 1. KERN-ENTITÄTEN (ENTITY RELATIONSHIP DIAGRAM)

```
┌─────────────┐         ┌──────────┐         ┌──────────┐
│    LEAD     │────────▶│  TASK    │────────▶│CUSTOMER  │
└─────────────┘         └──────────┘         └──────────┘
      │                      │                      │
      │                      │                      │
      ▼                      ▼                      ▼
┌─────────────┐         ┌──────────┐         ┌──────────┐
│  CONTACT    │         │ASSIGNMENT│         │  SERVICE │
└─────────────┘         └──────────┘         └──────────┘
```

### Entitäten im Überblick:
1. **LEAD** - Eingehende Anfrage (Telefon, E-Mail, Web)
2. **TASK** - Abgeleitete Aufgabe aus Lead
3. **CUSTOMER** - Kundenmaster (bestehend oder neu)
4. **SERVICE** - Brandschutz-Dienstleistung
5. **CONTACT** - Kontaktinformationen (kann mehrmals vorkommen)
6. **ASSIGNMENT** - Zuordnung Task → Mitarbeiter
7. **TIME_SLOT** - Verfügbare/gebuchte Zeitfenster

---

## 2. ENTITY DEFINITIONS & ATTRIBUTE KATALOG

### 2.1 LEAD
**Zweck:** Abbildung einer eingehenden Anfrage, unabhängig vom Kanal.

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
      "description": "Woher kam die Anfrage?"
    },
    "source_reference": {
      "type": "string",
      "required": false,
      "description": "Email-Message-ID, Telefon-Protokoll-ID, Form-Submission-ID"
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
      "pattern": "^[\\d+\\-() ]{7,20}$",
      "description": "Telefonnummer (international format)"
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
    "location_city": {
      "type": "string",
      "required": false,
      "description": "Stadt/Ort der Anfrage"
    },
    "initial_description": {
      "type": "text",
      "required": true,
      "description": "Freitextbeschreibung der Anfrage"
    },
    "service_category_raw": {
      "type": "string",
      "required": false,
      "description": "Unklassifizierter Service-Type (wie eingegeben)"
    },
    "urgency_indicator": {
      "type": "ENUM",
      "enum": ["normal", "soon", "urgent", "emergency"],
      "required": false,
      "description": "Zeitliche Dringlichkeit der Anfrage"
    },
    "lead_status": {
      "type": "ENUM",
      "enum": ["new", "classified", "assigned_to_task", "archived", "rejected"],
      "required": true,
      "default": "new",
      "description": "Status des Leads im Prozess"
    },
    "is_existing_customer": {
      "type": "boolean",
      "required": false,
      "description": "Ist bereits Kunde von A.RED?"
    },
    "linked_customer_id": {
      "type": "UUID",
      "required": false,
      "description": "Referenz zu existierendem Kunden (falls is_existing_customer=true)"
    },
    "classification_timestamp": {
      "type": "ISO8601_DateTime",
      "required": false,
      "description": "Wann wurde Lead klassifiziert?"
    },
    "classified_service_id": {
      "type": "UUID",
      "required": false,
      "description": "Service-ID nach Klassifikation (FK zu SERVICE)"
    },
    "classification_confidence": {
      "type": "float",
      "min": 0,
      "max": 1,
      "required": false,
      "description": "KI-Confidence für Klassifikation (0.0-1.0)"
    },
    "notes": {
      "type": "text",
      "required": false,
      "description": "Interne Notizen (z.B. Gründe für Ablehnung)"
    },
    "created_by": {
      "type": "string",
      "required": true,
      "description": "Benutzer, der Lead erfasst hat"
    },
    "created_at": {
      "type": "ISO8601_DateTime",
      "required": true,
      "description": "Timestamp der Erfassung"
    },
    "updated_at": {
      "type": "ISO8601_DateTime",
      "required": true,
      "description": "Letzter Änderungszeitpunkt"
    }
  }
}
```

---

### 2.2 CUSTOMER
**Zweck:** Kundenmaster - enthält sowohl bestehende als auch potenzielle neue Kunden.

```json
{
  "entity": "CUSTOMER",
  "primaryKey": "customer_id",
  "attributes": {
    "customer_id": {
      "type": "UUID",
      "required": true,
      "description": "Eindeutige Kunden-ID"
    },
    "customer_type": {
      "type": "ENUM",
      "enum": ["individual", "company"],
      "required": true,
      "description": "Privatperson oder Unternehmen?"
    },
    "status": {
      "type": "ENUM",
      "enum": ["prospect", "active", "inactive", "lost", "vip"],
      "required": true,
      "default": "prospect",
      "description": "Kundenstatus"
    },
    "name": {
      "type": "string",
      "required": true,
      "maxLength": 255,
      "description": "Name (Firmenname oder Vollname)"
    },
    "contact_person": {
      "type": "string",
      "required": false,
      "description": "Ansprechpartner (bei company_type=company)"
    },
    "street_address": {
      "type": "string",
      "required": false,
      "maxLength": 255
    },
    "city": {
      "type": "string",
      "required": false,
      "maxLength": 100
    },
    "postal_code": {
      "type": "string",
      "required": false,
      "pattern": "^\\d{5}$"
    },
    "country": {
      "type": "string",
      "required": false,
      "default": "DE",
      "description": "ISO 3166-1 alpha-2 code"
    },
    "primary_email": {
      "type": "email",
      "required": false
    },
    "primary_phone": {
      "type": "string",
      "required": false,
      "pattern": "^[\\d+\\-() ]{7,20}$"
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
      "description": "USt-IdNr. (für companies)"
    },
    "industry_sector": {
      "type": "string",
      "required": false,
      "description": "Branche (z.B. 'Handwerk', 'Retail', 'Logistik')"
    },
    "customer_lifetime_value": {
      "type": "float",
      "required": false,
      "description": "Gesamtumsatz des Kunden (EUR)"
    },
    "contract_start_date": {
      "type": "date",
      "required": false,
      "description": "Datum des ersten Vertrags"
    },
    "last_contact_date": {
      "type": "date",
      "required": false,
      "description": "Letzter Kontakt"
    },
    "preferred_communication": {
      "type": "ENUM",
      "enum": ["phone", "email", "in_person"],
      "required": false,
      "description": "Bevorzugte Kommunikationsart"
    },
    "external_id_filemaker": {
      "type": "string",
      "required": false,
      "description": "Kundennummer aus FileMaker (für Synchronisation)"
    },
    "notes": {
      "type": "text",
      "required": false,
      "description": "Allgemeine Notizen"
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

### 2.3 SERVICE
**Zweck:** Katalog der Brandschutz-Dienstleistungen (statischer Master-Daten).

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
      "maxLength": 255,
      "description": "Service-Name (z.B. 'Feuerlöscher-Prüfung')"
    },
    "category": {
      "type": "ENUM",
      "enum": [
        "fire_extinguisher_inspection",
        "fire_extinguisher_replacement",
        "fire_alarm_system_setup",
        "fire_alarm_system_maintenance",
        "emergency_exit_planning",
        "fire_safety_training",
        "risk_assessment",
        "compliance_audit",
        "other"
      ],
      "required": true,
      "description": "Service-Kategorie (Klassifikation)"
    },
    "description": {
      "type": "text",
      "required": true,
      "description": "Ausführliche Beschreibung"
    },
    "is_complex": {
      "type": "boolean",
      "required": true,
      "default": false,
      "description": "Erfordert manuelle Verhandlung?"
    },
    "automation_level": {
      "type": "ENUM",
      "enum": ["fully_automated", "semi_automated", "manual_only"],
      "required": true,
      "description": "Wie kann diese Service automatisiert werden?"
    },
    "typical_duration_minutes": {
      "type": "integer",
      "required": false,
      "description": "Typische Dauer in Minuten"
    },
    "requires_site_visit": {
      "type": "boolean",
      "required": true,
      "default": false,
      "description": "Muss vor Ort durchgeführt werden?"
    },
    "can_be_offered_immediately": {
      "type": "boolean",
      "required": true,
      "default": false,
      "description": "Kann sofort angeboten werden?"
    },
    "estimated_price_range_min": {
      "type": "float",
      "required": false,
      "description": "Geschätzter Mindestpreis (EUR)"
    },
    "estimated_price_range_max": {
      "type": "float",
      "required": false,
      "description": "Geschätzter Maximalpreis (EUR)"
    },
    "requires_certification": {
      "type": "boolean",
      "required": false,
      "description": "Braucht spezielle Zertifizierung?"
    },
    "external_id_filemaker": {
      "type": "string",
      "required": false,
      "description": "Service-Nummer aus FileMaker"
    },
    "is_active": {
      "type": "boolean",
      "required": true,
      "default": true,
      "description": "Wird diese Service noch angeboten?"
    },
    "keywords_for_matching": {
      "type": "array",
      "items": { "type": "string" },
      "required": false,
      "description": "Keywords für KI-basierte Klassifikation",
      "example": ["feuerlöscher", "prüfung", "wartung", "TÜV"]
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

### 2.4 TASK
**Zweck:** Konkrete Aufgabe, die aus einem Lead entsteht.

```json
{
  "entity": "TASK",
  "primaryKey": "task_id",
  "attributes": {
    "task_id": {
      "type": "UUID",
      "required": true
    },
    "source_lead_id": {
      "type": "UUID",
      "required": true,
      "description": "FK zu LEAD (woher kommt diese Aufgabe?)"
    },
    "task_type": {
      "type": "ENUM",
      "enum": ["new_service_inquiry", "follow_up", "scheduling", "proposal", "execution"],
      "required": true,
      "description": "Aufgaben-Typ"
    },
    "service_id": {
      "type": "UUID",
      "required": true,
      "description": "FK zu SERVICE (welche Service?)"
    },
    "customer_id": {
      "type": "UUID",
      "required": true,
      "description": "FK zu CUSTOMER (für wen?)"
    },
    "title": {
      "type": "string",
      "required": true,
      "maxLength": 255,
      "description": "Task-Titel"
    },
    "description": {
      "type": "text",
      "required": true,
      "description": "Aufgabendetails"
    },
    "priority": {
      "type": "ENUM",
      "enum": ["low", "medium", "high", "critical"],
      "required": true,
      "default": "medium"
    },
    "status": {
      "type": "ENUM",
      "enum": [
        "created",
        "waiting_for_info",
        "ready_to_schedule",
        "scheduled",
        "in_progress",
        "completed",
        "cancelled",
        "on_hold"
      ],
      "required": true,
      "default": "created",
      "description": "Aktueller Aufgabenstatus"
    },
    "deadline": {
      "type": "ISO8601_DateTime",
      "required": false,
      "description": "Wann muss es erledigt sein?"
    },
    "is_due_soon": {
      "type": "boolean",
      "required": false,
      "description": "Wird in KI-Alerts verwendet"
    },
    "assigned_to_employee_id": {
      "type": "string",
      "required": false,
      "description": "Wer ist verantwortlich? (FK zu EMPLOYEE)"
    },
    "assigned_at": {
      "type": "ISO8601_DateTime",
      "required": false
    },
    "estimated_effort_hours": {
      "type": "float",
      "required": false,
      "description": "Geschätzter Aufwand in Stunden"
    },
    "actual_effort_hours": {
      "type": "float",
      "required": false,
      "description": "Tatsächlicher Aufwand (nach Completion)"
    },
    "requires_site_visit": {
      "type": "boolean",
      "required": false,
      "description": "Muss vor Ort durchgeführt werden?"
    },
    "scheduled_date": {
      "type": "date",
      "required": false,
      "description": "Geplantes Durchführungsdatum"
    },
    "scheduled_time_slot_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu TIME_SLOT (wenn geplant)"
    },
    "is_blocking_existing_appointment": {
      "type": "boolean",
      "required": false,
      "default": false,
      "description": "KRITISCH: Verschiebt das einen bestehenden Kundenterm?"
    },
    "notes": {
      "type": "text",
      "required": false
    },
    "external_id_filemaker": {
      "type": "string",
      "required": false,
      "description": "Aufgabennummer aus FileMaker"
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
  }
}
```

---

### 2.5 TIME_SLOT
**Zweck:** Verfügbare und gebuchte Zeitfenster für Kundentermine.

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
      "description": "Verfügbar / Gebucht / Blockiert"
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
      "required": true,
      "description": "Dauer in Minuten"
    },
    "assigned_to_employee_id": {
      "type": "string",
      "required": false,
      "description": "Welcher Mitarbeiter?"
    },
    "assigned_task_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu TASK (falls booked)"
    },
    "customer_id": {
      "type": "UUID",
      "required": false,
      "description": "FK zu CUSTOMER (für wen?)"
    },
    "location": {
      "type": "string",
      "required": false,
      "description": "Ort des Termins (Adresse)"
    },
    "is_changeable": {
      "type": "boolean",
      "required": true,
      "default": false,
      "description": "Kann Termin noch verschoben werden?"
    },
    "recurring_rule": {
      "type": "RRULE",
      "required": false,
      "description": "Wenn wiederkehrend: iCalendar RRULE-Format"
    },
    "notes": {
      "type": "text",
      "required": false
    },
    "external_id_google_calendar": {
      "type": "string",
      "required": false,
      "description": "Google Calendar Event ID (für Sync)"
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

### 2.6 ASSIGNMENT (Task → Employee)
**Zweck:** Zuordnung von Aufgaben zu Mitarbeitern mit Gültigkeitsbereich.

```json
{
  "entity": "ASSIGNMENT",
  "primaryKey": ["task_id", "employee_id"],
  "attributes": {
    "task_id": {
      "type": "UUID",
      "required": true,
      "description": "FK zu TASK"
    },
    "employee_id": {
      "type": "string",
      "required": true,
      "description": "Mitarbeiter-ID"
    },
    "assigned_at": {
      "type": "ISO8601_DateTime",
      "required": true
    },
    "assigned_by": {
      "type": "string",
      "required": true,
      "description": "Wer hat zugeordnet?"
    },
    "role": {
      "type": "ENUM",
      "enum": ["primary_owner", "secondary", "reviewer"],
      "required": true,
      "description": "Rolle bei dieser Aufgabe"
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

### 2.7 CONTACT
**Zweck:** Kontaktinformationen (kann mehrmals für einen Kunden existieren).

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
    "contact_type": {
      "type": "ENUM",
      "enum": ["person", "department", "role_based"],
      "required": true
    },
    "name": {
      "type": "string",
      "required": true
    },
    "title": {
      "type": "string",
      "required": false,
      "description": "Titel/Position (z.B. 'Geschäftsführer')"
    },
    "email": {
      "type": "email",
      "required": false
    },
    "phone": {
      "type": "string",
      "required": false
    },
    "is_primary": {
      "type": "boolean",
      "required": true,
      "default": false,
      "description": "Haupt-Ansprechpartner?"
    },
    "preferred_communication": {
      "type": "ENUM",
      "enum": ["phone", "email", "sms", "whatsapp"],
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

## 3. DATENTYPEN & VALIDIERUNG

| Typ | Format | Beispiel | Validierung |
|-----|--------|---------|------------|
| **UUID** | RFC 4122 | `550e8400-e29b-41d4-a716-446655440000` | 36 chars, hex+hyphens |
| **ISO8601_DateTime** | ISO 8601 | `2026-05-14T19:30:00Z` | UTC timezone |
| **date** | ISO 8601 | `2026-05-14` | YYYY-MM-DD |
| **email** | RFC 5322 | `name@example.com` | Standard-Email-Validierung |
| **url** | RFC 3986 | `https://example.com` | Mit Protokoll |
| **ENUM** | Predefined set | siehe Entity-Def | Nur Werte aus der Liste |
| **RRULE** | iCalendar RFC 5545 | `FREQ=WEEKLY;BYDAY=MO,WE,FR` | Google Calendar kompatibel |

---

## 4. KRITISCHE GESCHÄFTSREGELN (IN MODELLEN)

### 4.1 Kundentermine (TIME_SLOT) - SCHUTZ
- ✅ `is_changeable=false` → Task kann diesen Termin NICHT verschieben
- ✅ Vor Scheduling: Check ob `is_blocking_existing_appointment=true`
- ✅ Neue Termine dürfen nur in verfügbaren Slots gebucht werden (`slot_type="available"`)

### 4.2 Komplexe Services - KEINE AUTOMATION
- Services mit `is_complex=true` & `automation_level="manual_only"`:
  - Können NICHT automatisch angeboten werden
  - Müssen immer manuell verhandelt werden
  - Können NICHT in Automated Suggestions landen

### 4.3 Lead-Klassifikation - KI-VORSCHLAG NUR
- `classification_confidence < 0.7` → Lead-Status bleibt auf `new`, KI-Vorschlag wird nur als Notification angezeigt
- Manuelle Genehmigung erforderlich, bevor `lead_status="classified"` gesetzt wird

### 4.4 Existierende Kunden - PRIORITÄT
- `is_existing_customer=true` → `status` auf `active` bei Kunden setzen
- Bestehende Kunden kriegen Priorität in Task-Zuordnung

---

## 5. RELATIONEN & FOREIGN KEYS

```
LEAD
  ├─ FK customer_id → CUSTOMER (optional, wenn bekannt)
  └─ FK classified_service_id → SERVICE (nach KI-Klassifikation)

TASK
  ├─ FK source_lead_id → LEAD (required)
  ├─ FK service_id → SERVICE (required)
  ├─ FK customer_id → CUSTOMER (required)
  ├─ FK assigned_to_employee_id → EMPLOYEE (optional)
  └─ FK scheduled_time_slot_id → TIME_SLOT (optional, wenn geplant)

ASSIGNMENT
  ├─ FK task_id → TASK
  └─ FK employee_id → EMPLOYEE

TIME_SLOT
  ├─ FK assigned_task_id → TASK (optional)
  ├─ FK customer_id → CUSTOMER (optional)
  └─ FK assigned_to_employee_id → EMPLOYEE

CONTACT
  └─ FK customer_id → CUSTOMER (required)
```

---

## 6. INTEGRATIONS-PUNKTE (ZU FILESMAKER & GOOGLE WORKSPACE)

### 6.1 FileMaker-Sync
Folgende Felder sind Sync-relevant:
- `CUSTOMER.external_id_filemaker` ← Master-Kundennummer
- `SERVICE.external_id_filemaker` ← Master-Service-Nummer
- `TASK.external_id_filemaker` ← Master-Aufgabennummer
- `TIME_SLOT.external_id_google_calendar` ← Google Calendar Event ID

### 6.2 Google Workspace Integration
- Google Calendar: TIME_SLOT ↔ Calendar Events (bidirektional)
- Gmail: Lead Source Reference für Email-Rückverfolgung
- Google Sheets: Tägliche Lead/Task-Reports (optional)

---

## 7. DATENQUALITÄTS-ANFORDERUNGEN

| Entität | Minimum-Felder | Abhängigkeiten |
|---------|----------------|-----------------|
| **LEAD** | source_channel, received_date, contact_person_name, initial_description | Keine (unabhängig) |
| **CUSTOMER** | name, customer_type | Kann ohne Lead existieren |
| **TASK** | source_lead_id, service_id, customer_id, title, description | Benötigt existierende LEAD, SERVICE, CUSTOMER |
| **TIME_SLOT** | start_datetime, end_datetime, slot_type | Optional: assigned_task_id, customer_id |
| **SERVICE** | name, category, description, is_complex, automation_level | Keine (statisch) |

---

## 8. BEISPIEL-DATENSÄTZE

### Beispiel 1: Neuer Lead (Phone-Kanal)
```json
{
  "lead_id": "550e8400-e29b-41d4-a716-446655440001",
  "source_channel": "phone",
  "received_date": "2026-05-14T10:30:00Z",
  "contact_person_name": "Klaus Schmidt",
  "contact_phone": "+49 30 12345678",
  "company_name": "Schmidt Bürobedarf GmbH",
  "initial_description": "Benötigen Feuerlöscher-Inspektion für zwei Standorte",
  "urgency_indicator": "normal",
  "lead_status": "new",
  "is_existing_customer": true,
  "classified_service_id": null,
  "classification_confidence": null,
  "created_by": "anna.mueller",
  "created_at": "2026-05-14T10:35:00Z",
  "updated_at": "2026-05-14T10:35:00Z"
}
```

### Beispiel 2: Klassifizierter Lead → Task
```json
{
  "lead_id": "550e8400-e29b-41d4-a716-446655440001",
  "lead_status": "assigned_to_task",
  "classified_service_id": "e29b-41d4-550e-8400-a716446655440002",
  "classification_confidence": 0.92,
  "classification_timestamp": "2026-05-14T10:45:00Z"
}
```

### Beispiel 3: Task mit Schedule-Schutz
```json
{
  "task_id": "a716-446655440003-550e8400-e29b-41d4",
  "source_lead_id": "550e8400-e29b-41d4-a716-446655440001",
  "task_type": "scheduling",
  "service_id": "e29b-41d4-550e-8400-a716446655440002",
  "customer_id": "550e-8400-e29b-41d4-a716446655440004",
  "title": "Feuerlöscher-Inspektion - 2 Standorte",
  "status": "ready_to_schedule",
  "is_blocking_existing_appointment": false,
  "scheduled_date": "2026-05-21",
  "assigned_to_employee_id": "emp_003",
  "requires_site_visit": true
}
```

---

## 9. NEXT STEPS

1. ✅ **Datenmodelle definiert** (diese Datei)
2. 📋 **→ JSON-Schemas** (separate Dateien unter `/schemas/`)
3. 📋 **→ Geschäftsregeln** (doc/3_GESCHAEFTREGELN.md)
4. 📋 **→ Integration-Plan** (doc/4_INTEGRATIONS-PLAN.md)

---

**Erstellt:** 2026-05-14  
**Version:** 1.0  
**Status:** DRAFT (warten auf Feedback)
