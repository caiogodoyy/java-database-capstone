## MySQL Database Design

> Notes
>
> * MySQL 8.0+ (for `CHECK` constraints).
> * Prefer **soft deletes** (`is_active`, `deleted_at`) to preserve history.
> * Time is stored in UTC; convert at the edges.
> * Overlap prevention for appointments is enforced in the **service layer**; the DB ensures basics (no exact duplicates, valid ranges).

```sql
-- =====================================================================
-- SCHEMA: smart_clinic
-- =====================================================================
CREATE DATABASE IF NOT EXISTS smart_clinic CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE smart_clinic;

-- -------------------------------------------------
-- 1) patients
-- -------------------------------------------------
CREATE TABLE patients (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  first_name      VARCHAR(100)     NOT NULL,
  last_name       VARCHAR(100)     NOT NULL,
  email           VARCHAR(255)     NOT NULL,
  phone           VARCHAR(32)      NULL, -- format validated in app code
  dob             DATE             NULL,
  gender          ENUM('Male','Female','Other','PreferNotToSay') NULL,
  created_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  is_active       TINYINT(1)       NOT NULL DEFAULT 1,
  deleted_at      DATETIME         NULL,
  UNIQUE KEY uk_patients_email (email)
) ENGINE=InnoDB;

-- -------------------------------------------------
-- 2) doctors
-- -------------------------------------------------
CREATE TABLE doctors (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  first_name      VARCHAR(100)     NOT NULL,
  last_name       VARCHAR(100)     NOT NULL,
  email           VARCHAR(255)     NOT NULL,
  phone           VARCHAR(32)      NULL,
  specialization  VARCHAR(120)     NOT NULL,
  license_number  VARCHAR(80)      NOT NULL,
  bio             TEXT             NULL,
  created_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  is_active       TINYINT(1)       NOT NULL DEFAULT 1,
  deleted_at      DATETIME         NULL,
  UNIQUE KEY uk_doctors_email (email),
  UNIQUE KEY uk_doctors_license (license_number)
) ENGINE=InnoDB;

-- -------------------------------------------------
-- 3) admins
-- -------------------------------------------------
CREATE TABLE admins (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  username        VARCHAR(100)     NOT NULL,
  email           VARCHAR(255)     NOT NULL,
  password_hash   VARCHAR(255)     NOT NULL, -- use strong hashing (BCrypt/Argon2)
  role            ENUM('SUPERADMIN','ADMIN','ANALYST') NOT NULL DEFAULT 'ADMIN',
  last_login_at   DATETIME         NULL,
  created_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  is_active       TINYINT(1)       NOT NULL DEFAULT 1,
  UNIQUE KEY uk_admins_username (username),
  UNIQUE KEY uk_admins_email (email)
) ENGINE=InnoDB;

-- -------------------------------------------------
-- 4) clinic_locations (optional but practical)
-- -------------------------------------------------
CREATE TABLE clinic_locations (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  name            VARCHAR(150)     NOT NULL,
  address_line1   VARCHAR(200)     NOT NULL,
  address_line2   VARCHAR(200)     NULL,
  city            VARCHAR(120)     NOT NULL,
  state_region    VARCHAR(120)     NULL,
  postal_code     VARCHAR(40)      NULL,
  country         VARCHAR(80)      NOT NULL,
  phone           VARCHAR(32)      NULL,
  timezone        VARCHAR(64)      NOT NULL DEFAULT 'UTC',
  is_active       TINYINT(1)       NOT NULL DEFAULT 1
) ENGINE=InnoDB;

-- -------------------------------------------------
-- 5) appointments
--   - One-hour bookings are enforced in the service layer; DB validates ranges.
--   - Soft delete by status or separate audit tables if needed.
-- -------------------------------------------------
CREATE TABLE appointments (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  doctor_id       BIGINT UNSIGNED  NOT NULL,
  patient_id      BIGINT UNSIGNED  NOT NULL,
  location_id     BIGINT UNSIGNED  NULL, -- null for telehealth
  start_time_utc  DATETIME         NOT NULL,
  end_time_utc    DATETIME         NOT NULL,
  status          ENUM('SCHEDULED','COMPLETED','CANCELLED') NOT NULL DEFAULT 'SCHEDULED',
  reason          VARCHAR(255)     NULL,
  notes           TEXT             NULL,
  created_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  CONSTRAINT fk_appt_doctor   FOREIGN KEY (doctor_id)  REFERENCES doctors(id)   ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT fk_appt_patient  FOREIGN KEY (patient_id) REFERENCES patients(id)  ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT fk_appt_location FOREIGN KEY (location_id) REFERENCES clinic_locations(id) ON DELETE SET NULL ON UPDATE CASCADE,

  CONSTRAINT chk_appt_window CHECK (end_time_utc > start_time_utc),

  -- Prevents exact duplicate time slots for same doctor; overlapping windows are handled in services
  UNIQUE KEY uk_doctor_exact_slot (doctor_id, start_time_utc, end_time_utc),

  -- Useful lookups
  KEY idx_appt_doctor_time (doctor_id, start_time_utc),
  KEY idx_appt_patient_time (patient_id, start_time_utc)
) ENGINE=InnoDB;

-- -------------------------------------------------
-- 6) doctor_unavailability
--   - Doctors can block time; patient booking UI excludes these intervals.
-- -------------------------------------------------
CREATE TABLE doctor_unavailability (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  doctor_id       BIGINT UNSIGNED  NOT NULL,
  start_time_utc  DATETIME         NOT NULL,
  end_time_utc    DATETIME         NOT NULL,
  reason          VARCHAR(255)     NULL, -- e.g., vacation, conference
  created_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT fk_unavail_doctor FOREIGN KEY (doctor_id) REFERENCES doctors(id) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT chk_unavail_window CHECK (end_time_utc > start_time_utc),
  KEY idx_unavail_doctor_time (doctor_id, start_time_utc)
) ENGINE=InnoDB;

-- -------------------------------------------------
-- 7) payments (optional; ties to appointments)
-- -------------------------------------------------
CREATE TABLE payments (
  id              BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  appointment_id  BIGINT UNSIGNED  NOT NULL,
  patient_id      BIGINT UNSIGNED  NOT NULL,
  amount_cents    INT UNSIGNED     NOT NULL,
  currency        CHAR(3)          NOT NULL DEFAULT 'USD',
  method          ENUM('CASH','CARD','INSURANCE','ONLINE') NOT NULL,
  status          ENUM('PENDING','PAID','REFUNDED','FAILED') NOT NULL DEFAULT 'PENDING',
  external_ref    VARCHAR(100)     NULL, -- PSP transaction id
  created_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  CONSTRAINT fk_payment_appt   FOREIGN KEY (appointment_id) REFERENCES appointments(id) ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT fk_payment_patient FOREIGN KEY (patient_id)    REFERENCES patients(id)     ON DELETE RESTRICT ON UPDATE CASCADE,
  KEY idx_payments_patient_time (patient_id, created_at)
) ENGINE=InnoDB;

-- -------------------------------------------------
-- Stored procedure placeholder for monthly stats (admin report)
-- -------------------------------------------------
DELIMITER $$
CREATE PROCEDURE sp_monthly_appointments(IN p_year INT)
BEGIN
  SELECT
    YEAR(start_time_utc) AS year,
    MONTH(start_time_utc) AS month,
    COUNT(*) AS total
  FROM appointments
  WHERE YEAR(start_time_utc) = p_year
  GROUP BY YEAR(start_time_utc), MONTH(start_time_utc)
  ORDER BY month;
END$$
DELIMITER ;
```

### Design choices & constraints (quick justifications)

* **Soft delete vs cascade delete:** We keep patients/doctors historical data; FK uses `ON DELETE RESTRICT`. Disable instead of delete to maintain appointment history.
* **Overlapping appointments:** Complex to enforce purely in SQL; handled in the **service layer** using conflict checks against `appointments` and `doctor_unavailability`. DB enforces valid windows and prevents exact duplicates.
* **Indexing:** Composite indexes support common queries (doctor’s calendar, patient’s history).
* **Email/phone validation:** Done in application layer; DB ensures **uniqueness** where appropriate.
* **Time zones:** Store UTC in DB; convert for UI using location/timezone.

## MongoDB Collection Design

We’ll model **`prescriptions`** in MongoDB to allow flexible structures (variable medication lists, evolving metadata). Each prescription links back to MySQL entities via IDs, but carries rich, nested content.

### Collection: `prescriptions`

```json
{
  "_id": { "$oid": "66c8f1a4b9a0f42e1f123456" },
  "schemaVersion": 2,
  "appointmentId": 1023,              // MySQL appointments.id
  "patient": {
    "id": 5012,                        // MySQL patients.id (for joins via app)
    "name": "Ana Rodrigues"
  },
  "doctor": {
    "id": 204,                         // MySQL doctors.id
    "name": "Dr. Marcos Silva",
    "specialization": "Cardiology"
  },
  "issuedAt": { "$date": "2025-08-15T13:20:00Z" },
  "medications": [
    {
      "name": "Atorvastatin",
      "dosage": "20 mg",
      "route": "oral",
      "frequency": "once daily",
      "durationDays": 30,
      "notes": "Take at night; monitor for muscle pain."
    },
    {
      "name": "Aspirin",
      "dosage": "81 mg",
      "route": "oral",
      "frequency": "once daily",
      "durationDays": 30
    }
  ],
  "instructions": "Maintain low-fat diet, regular exercise. Return in 4 weeks.",
  "allergiesChecked": true,
  "refills": {
    "allowed": 1,
    "remaining": 1,
    "history": [
      {
        "refillAt": { "$date": "2025-09-15T09:00:00Z" },
        "pharmacy": { "name": "Farmácia Boa Saúde", "location": "Recife - Centro" }
      }
    ]
  },
  "pharmacyPreference": {
    "name": "Farmácia Boa Saúde",
    "location": "Recife - Centro",
    "contact": "+55 81 3333-4444"
  },
  "tags": ["cardio", "cholesterol", "follow-up"],
  "attachments": [
    {
      "type": "pdf",
      "url": "s3://prescriptions/1023.pdf",
      "uploadedAt": { "$date": "2025-08-15T13:21:10Z" }
    }
  ],
  "audit": {
    "createdBy": "system",
    "createdAt": { "$date": "2025-08-15T13:20:00Z" },
    "updatedAt": { "$date": "2025-08-15T13:21:10Z" }
  }
}
```

### Why this works well in MongoDB

* **Flexible shape:** Different visits can have different `medications`, `attachments`, or `tags` without schema churn.
* **Embedded structures:** Patient/doctor names are embedded for snapshotting; numeric IDs reference authoritative rows in MySQL.
* **Versioning:** `schemaVersion` supports non-breaking evolution.
* **Auditing:** Embedded `audit` enables quick traceability.
