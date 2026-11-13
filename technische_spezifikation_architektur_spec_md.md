# SPEC.md – Hausverwaltungs-PWA (4 Wohneinheiten / 3 Eigeneinheiten)

## 1. Projektziel & Systemübersicht
Entwicklung einer selbstgehosteten **Progressive Web App (PWA)**, die sich unter Windows 10/11 (via Edge/Chrome) als eigenständige Desktop-Anwendung installieren lässt und mobil auf Smartphones nutzbar ist.

### Objekt-Konstellation (Kernanforderung)
* **Gesamtobjekt:** Mehrfamilienhaus mit **4 Wohneinheiten (WE 1 – WE 4)**.
* **Eigentumsverhältnisse:**
  * **WE 1, WE 2, WE 3:** Im eigenen Bestand, vermietet an drei Mietparteien (Einnahmen- & Mietverwaltung + Betriebskostenabrechnung nach BetrKV + Steuer-Export Anlage V).
  * **WE 4:** Fremdeigentümer (anteilige Gesamthauskosten-Abrechnung / Hausgeldabrechnung nach Miteigentumsanteilen & Zählern).
* **Cloud-Integration:** Vollständige Datenhaltung und Belegarchivierung über **Google Cloud (Firebase + Google Drive API v3)** sowie automatische Rechnungserkennung über die **Gemini API**.

---

## 2. Tech-Stack & Windows-Desktop-PWA Setup

* **Frontend:** React 19, TypeScript, Vite, TailwindCSS, Lucide Icons
* **PWA-Engine:** `vite-plugin-pwa` mit Service Worker (Offline-Caching der Stammdaten)
* **Backend / Datenbank:** Firebase Authentication (Google Sign-In) & Cloud Firestore
* **Dateispeicher:** Google Drive API v3 (OAuth 2.0 Scope: `https://www.googleapis.com/auth/drive.file`)
* **KI-Belegauslesung:** Google Gemini API (`gemini-2.5-flash` mit Structured JSON Output)
* **PDF-Erstellung:** `@react-pdf/renderer` (clientseitige Generierung ohne Serverkosten)

### Windows Desktop Manifest (`manifest.webmanifest`)
```json
{
  "name": "HausPilot Cloud – Immobilienverwaltung",
  "short_name": "HausPilot",
  "start_url": "/",
  "display": "standalone",
  "display_override": ["window-controls-overlay", "standalone"],
  "background_color": "#0f172a",
  "theme_color": "#2563eb",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

---

## 3. Google Cloud & Drive Architektur

### Automatische Google Drive Ordnerstruktur
Beim ersten Login legt die App über die Google Drive API folgende Struktur an und speichert die `folderId`s in Firestore:
```text
📁 Hausverwaltung_Gesamtobjekt/
 ├── 📁 00_Inbox_Belege/              <-- Drag & Drop Upload / Handy-Scan
 ├── 📁 01_Stammdaten_Vertraege/      <-- Mietverträge WE 1-3, Teilungserklärung
 └── 📁 2026/
      ├── 📁 Rechnungen_Gesamthaus/   <-- Grundsteuer, Versicherung, Heizung, Allgemeinstrom
      ├── 📁 Rechnungen_Direkt/       <-- Einzelrechnungen nur für WE 1, WE 2, WE 3 oder WE 4
      ├── 📁 Zaehlerstaende/          <-- Fotos der Zählerstände zum Stichtag
      └── 📁 Abrechnungen_PDF/        <-- Generierte NK-Abrechnungen (WE 1-3) & WEG-Abrechnung (WE 4)
```

### Automatisierter Beleg-Workflow (OCR -> Drive -> Firestore)
1. Nutzer zieht PDF-Rechnung oder Foto in die PWA.
2. Client sendet Datei an Gemini API (`gemini-2.5-flash`) mit System-Prompt zur Extraktion von: `vendor`, `invoiceDate`, `servicePeriodStart`, `servicePeriodEnd`, `grossAmount`, `category`, `isApportionable` (BetrKV-Check), `sec35aLaborAmount` (Lohnanteil für § 35a EStG) und `suggestedAllocationKey`.
3. Nutzer bestätigt/korrigiert die Vorschau mit einem Klick.
4. App benennt die Datei standardisiert um (`YYYY-MM-DD_Kategorie_Betrag.pdf`), lädt sie in den passenden Google-Drive-Ordner hoch und speichert den Datensatz inklusive `driveWebViewLink` in Firestore.

---

## 4. Datenmodell (TypeScript Interfaces)

```typescript
export type UnitId = 'WE1' | 'WE2' | 'WE3' | 'WE4';
export type AllocationKey = 'MEA' | 'SQM' | 'METER_WATER' | 'METER_HEAT' | 'EQUAL' | 'DIRECT';

export interface Unit {
  id: UnitId;
  label: string;              // z.B. "EG Links (WE 1)"
  isOwnedByUser: boolean;     // true für WE1-WE3, false für WE4
  sqm: number;                // Wohnfläche in m²
  mea: number;                // Miteigentumsanteil (z.B. 260 von 1000)
  personCount: number;        // Personenanzahl
}

export interface TenantOrOwner {
  id: string;
  unitId: UnitId;
  role: 'TENANT' | 'EXTERNAL_OWNER';
  name: string;
  email: string;
  moveInDate: string;         // ISO-Datum (für zeitanteilige Berechnung)
  moveOutDate?: string;
  monthlyColdRent: number;    // 0 bei WE4
  monthlyUtilityAdvance: number; // NK-Vorauszahlung (WE1-3) bzw. Hausgeld-Vorauszahlung (WE4)
  ibanLast4?: string;         // Für automatischen CSV-Bankabgleich
}

export interface Expense {
  id: string;
  billingYear: number;
  title: string;
  vendor: string;
  invoiceDate: string;
  grossAmount: number;
  allocationKey: AllocationKey;
  directUnitId?: UnitId;      // Pflichtfeld, falls allocationKey === 'DIRECT'
  isApportionable: boolean;   // true = umlagefähig auf Mieter (BetrKV), false = Eigentümerlast
  sec35aLaborAmount: number;  // Handwerkerlohnanteil nach § 35a EStG
  driveFileId?: string;
  driveUrl?: string;
}

export interface MeterReading {
  id: string;
  billingYear: number;
  meterType: 'COLD_WATER' | 'WARM_WATER' | 'HEAT';
  unitId: UnitId | 'MAIN';
  startValue: number;
  endValue: number;
  unitSymbol: 'm³' | 'kWh' | 'MWh';
}

export interface Payment {
  id: string;
  unitId: UnitId;
  date: string;
  month: string;              // "2026-01"
  amount: number;
  type: 'RENT_AND_NK' | 'HAUSGELD_WE4';
  status: 'PAID' | 'PARTIAL' | 'OPEN';
}
```

---

## 5. Zweistufige Berechnungslogik (Abrechnungs-Engine)

### Stufe 1: Gesamthaus-Verteilung auf alle 4 Einheiten
Für jede Ausgabe $e$ des Abrechnungsjahres wird der Anteil je Einheit $i \in \{\text{WE1}, \text{WE2}, \text{WE3}, \text{WE4}\}$ bestimmt:

1. **Nach Miteigentumsanteilen (`MEA`):**
   $$C_{e,i} = B_e \cdot \frac{\text{MEA}_i}{\sum_{k=1}^{4} \text{MEA}_k}$$
2. **Nach Wohnfläche (`SQM`):**
   $$C_{e,i} = B_e \cdot \frac{\text{qm}_i}{\sum_{k=1}^{4} \text{qm}_k}$$
3. **Nach Zählerverbrauch (`METER_WATER` / `METER_HEAT`):**
   $$C_{e,i} = B_e \cdot \frac{V_i}{\sum_{k=1}^{4} V_k} \quad \text{mit } V_i = Z_{\text{Ende},i} - Z_{\text{Start},i}$$
4. **Direktzuordnung (`DIRECT`):**
   $$C_{e,i} = \begin{cases} B_e & \text{wenn } i = \text{directUnitId} \\ 0 & \text{sonst} \end{cases}$$

### Stufe 2A: Mieter-Nebenkostenabrechnung (nur WE 1, WE 2, WE 3)
Für jede eigene Einheit $i \in \{\text{WE1}, \text{WE2}, \text{WE3}\}$ werden **ausschließlich umlagefähige Positionen** (`isApportionable === true`) summiert und bei unterjährigem Mieterwechsel zeitanteilig ($\frac{\text{Miettage}}{365}$) gewichtet:

$$\text{Gesamtkosten}_{\text{Mieter}, i} = \sum_{e \in E_{\text{umlagefähig}}} C_{e,i} \cdot \frac{d_i}{365}$$

$$\text{Saldo}_{\text{Mieter}, i} = \text{Gesamtkosten}_{\text{Mieter}, i} - \sum_{m=1}^{12} \text{NK-Vorauszahlung}_{i,m}$$
*(Positiver Saldo = Nachzahlung durch Mieter; negativer Saldo = Guthaben für Mieter)*

### Stufe 2B: Eigentümer-Abrechnung für WE 4 (Fremdeigentümer)
Für WE 4 fließen **alle** Gesamthauskosten (umlagefähige Betriebskosten + nicht umlagefähige Hauskosten wie Gebäudeinstandhaltung, Rücklagen, Allgemeinkosten) ein:

$$\text{Saldo}_{\text{WE4}} = \sum_{e \in E_{\text{Gesamthaus}}} C_{e,\text{WE4}} - \sum_{m=1}^{12} \text{Hausgeld-Zahlung}_{\text{WE4},m}$$

### Stufe 3: Steuer-Auswertung (Anlage V für WE 1–3)
* **Einnahmen:** Summe aller eingegangenen Kaltmieten + NK-Vorauszahlungen von WE 1–3.
* **Werbungskosten:** Anteilige Gesamthauskosten (umlagefähig + nicht umlagefähig) für WE 1–3 + direkte Reparatur-/Verwaltungskosten für WE 1–3.

---

## 6. Implementierungs-Roadmap
1. **Phase 1 (Setup):** `npm create vite@latest hauspilot -- --template react-ts`, Installation von `tailwindcss`, `lucide-react`, `firebase`, `vite-plugin-pwa`.
2. **Phase 2 (Auth & Drive):** Google Cloud Console Projekt anlegen, Firebase Auth aktivieren, Google Drive API v3 freischalten und Ordner-Initialisierung bauen.
3. **Phase 3 (CRUD & OCR):** Stammdaten-Maske (4 Einheiten), Beleg-Upload mit `gemini-2.5-flash` JSON-Extraktion und CSV-Bankimport (MT940 / Camt.053 / Standard-CSV).
4. **Phase 4 (Engine & PDF):** Berechnungsmodul gemäß Abschnitt 5 implementieren und PDF-Export mit `@react-pdf/renderer` (inkl. § 35a EStG Tabelle) anbinden.