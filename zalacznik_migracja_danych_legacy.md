# Załącznik - model danych legacy i zakres migracji

## 1. Cel załącznika

Niniejszy załącznik definiuje referencyjny, roboczy model danych obecnego systemu legacy dla potrzeb przygotowania oferty, estymacji prac migracyjnych oraz zaprojektowania mechanizmu przeniesienia danych do nowego rozwiązania.

Dokument nie stanowi fizycznego zrzutu produkcyjnej bazy danych, lecz opisuje oczekiwany zakres danych źródłowych, ich relacje oraz minimalny poziom odwzorowania, jaki wykonawca powinien uwzględnić w ofercie i analizie przedwdrożeniowej.

---

## 2. Charakterystyka systemu źródłowego

Na potrzeby RFP należy przyjąć następujące założenia dotyczące obecnego rozwiązania:

- system źródłowy jest aplikacją monolityczną z relacyjną bazą danych,
- część integracji zewnętrznych zapisuje dane asynchronicznie, co oznacza możliwość występowania opóźnień i częściowo niespójnych statusów,
- część historycznych rekordów została utworzona w starszych wersjach systemu i może nie zawierać pełnego zestawu pól,
- identyfikatory techniczne w systemie legacy są liczbowe i lokalne dla tabel,
- część danych referencyjnych jest utrzymywana słownikowo, a część w postaci wartości tekstowych,
- załączniki są przechowywane poza bazą danych, natomiast w bazie znajdują się ich metadane i ścieżki dostępu.

---

## 3. Główne encje objęte migracją

Minimalny zakres migracji powinien objąć:

1. użytkowników, klientów i dane kontaktowe,
2. konta pracowników, role i uprawnienia podstawowe,
3. urządzenia, modele i kategorie sprzętu,
4. zgłoszenia serwisowe oraz ich statusy,
5. historię klasyfikacji software/hardware,
6. komunikację z klientem i notatki wewnętrzne,
7. przesyłki przychodzące i zwrotne wraz z historią statusów,
8. wyniki diagnostyki, wyceny i decyzje klienta,
9. naprawy, czynności serwisowe i wykorzystane części,
10. płatności i dokumenty rozliczeniowe,
11. załączniki i dokumenty sprawy,
12. logi audytowe wymagane operacyjnie.

---

## 4. Referencyjny wykaz tabel systemu legacy

### 4.1 Tabele użytkowników i dostępu

#### `users`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz główny użytkownika | Zachować jako `legacy_user_id` |
| `user_type` | varchar(20) | `customer`, `employee`, `admin` | Mapować do modelu tożsamości docelowej |
| `email` | varchar(255) | Adres e-mail | Walidacja unikalności |
| `phone` | varchar(50) | Numer telefonu | Normalizacja formatu |
| `password_hash` | varchar(255) | Hash hasła | Migracja tylko jeśli zgodna z polityką bezpieczeństwa |
| `status` | varchar(30) | Status konta | Mapowanie do statusów docelowych |
| `created_at` | datetime | Data utworzenia | Zachować |
| `updated_at` | datetime | Data aktualizacji | Zachować |
| `last_login_at` | datetime null | Ostatnie logowanie | Opcjonalnie |

#### `customer_profiles`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz profilu klienta | Zachować jako `legacy_customer_id` |
| `user_id` | bigint FK -> users.id | Powiązanie z kontem | Relacja obowiązkowa |
| `customer_no` | varchar(50) | Numer klienta | Zachować w polu referencyjnym |
| `first_name` | varchar(120) | Imię | Zachować |
| `last_name` | varchar(120) | Nazwisko | Zachować |
| `company_name` | varchar(255) null | Nazwa firmy | Opcjonalnie |
| `tax_id` | varchar(50) null | NIP / identyfikator firmy | Walidacja biznesowa |
| `preferred_contact_channel` | varchar(30) | Preferowany kanał kontaktu | Mapowanie słownika |
| `gdpr_consent` | bit | Zgoda na przetwarzanie danych | Zachować |
| `marketing_consent` | bit | Zgoda marketingowa | Opcjonalnie |

#### `employee_profiles`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz profilu pracownika | Zachować jako `legacy_employee_id` |
| `user_id` | bigint FK -> users.id | Powiązanie z kontem | Relacja obowiązkowa |
| `employee_no` | varchar(50) | Numer pracownika | Zachować |
| `department` | varchar(100) | Dział | Mapowanie słownikowe |
| `job_title` | varchar(100) | Stanowisko | Opcjonalnie |
| `is_active` | bit | Aktywność pracownika | Zachować |

#### `roles`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz roli | Zachować jako referencję legacy |
| `code` | varchar(50) | Kod roli | Mapować do ról docelowych |
| `name` | varchar(100) | Nazwa roli | Zachować opisowo |

#### `user_roles`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz relacji | Techniczne |
| `user_id` | bigint FK -> users.id | Użytkownik | Obowiązkowe mapowanie |
| `role_id` | bigint FK -> roles.id | Rola | Obowiązkowe mapowanie |
| `assigned_at` | datetime | Data przypisania | Opcjonalnie |

### 4.2 Tabele urządzeń i danych referencyjnych

#### `device_categories`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz kategorii | Mapowanie do słownika docelowego |
| `code` | varchar(50) | Kod kategorii | Np. `LAPTOP`, `PHONE`, `INDUSTRIAL` |
| `name` | varchar(100) | Nazwa kategorii | Zachować |

#### `device_models`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz modelu | Zachować jako `legacy_device_model_id` |
| `manufacturer` | varchar(120) | Producent | Normalizacja producentów |
| `model_name` | varchar(150) | Model | Zachować |
| `category_id` | bigint FK -> device_categories.id | Kategoria | Mapowanie relacji |
| `is_active` | bit | Aktywność modelu | Opcjonalnie |

#### `devices`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz urządzenia | Zachować jako `legacy_device_id` |
| `customer_id` | bigint FK -> customer_profiles.id | Właściciel urządzenia | Relacja obowiązkowa |
| `device_model_id` | bigint FK -> device_models.id | Model urządzenia | Relacja obowiązkowa |
| `serial_number` | varchar(120) | Numer seryjny | Klucz biznesowy, walidacja duplikatów |
| `asset_tag` | varchar(120) null | Oznaczenie majątkowe | Opcjonalnie |
| `purchase_date` | date null | Data zakupu | Opcjonalnie |
| `warranty_until` | date null | Koniec gwarancji | Opcjonalnie |
| `device_status` | varchar(30) | Ogólny status urządzenia | Mapowanie słownika |
| `created_at` | datetime | Data rejestracji | Zachować |

### 4.3 Tabele zgłoszeń i procesów serwisowych

#### `service_requests`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz zgłoszenia | Zachować jako `legacy_request_id` |
| `request_no` | varchar(50) | Numer zgłoszenia | Zachować i eksponować w nowym systemie |
| `customer_id` | bigint FK -> customer_profiles.id | Klient | Relacja obowiązkowa |
| `device_id` | bigint FK -> devices.id | Urządzenie | Relacja obowiązkowa |
| `created_by_user_id` | bigint FK -> users.id | Autor zgłoszenia | Zachować |
| `issue_type` | varchar(20) | `software` lub `hardware` | Mapowanie obowiązkowe |
| `subject` | varchar(255) | Krótki temat | Zachować |
| `description` | text | Opis problemu | Zachować |
| `priority` | varchar(20) | Priorytet | Mapowanie do modelu docelowego |
| `current_status` | varchar(50) | Bieżący status | Zachować jako status operacyjny |
| `assigned_employee_id` | bigint FK -> employee_profiles.id null | Aktualnie przypisany pracownik | Opcjonalnie |
| `created_at` | datetime | Data utworzenia | Zachować |
| `closed_at` | datetime null | Data zamknięcia | Opcjonalnie |

#### `request_status_history`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz wpisu statusu | Techniczne |
| `service_request_id` | bigint FK -> service_requests.id | Zgłoszenie | Relacja obowiązkowa |
| `status_code` | varchar(50) | Kod statusu | Zachować historię |
| `substatus_code` | varchar(50) null | Podstatus | Opcjonalnie |
| `changed_by_user_id` | bigint FK -> users.id null | Autor zmiany | Opcjonalnie |
| `changed_at` | datetime | Czas zmiany | Zachować |
| `comment` | varchar(500) null | Komentarz | Opcjonalnie |

#### `remote_support_sessions`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz sesji | Zachować dla spraw software |
| `service_request_id` | bigint FK -> service_requests.id | Zgłoszenie | Relacja obowiązkowa |
| `consultant_employee_id` | bigint FK -> employee_profiles.id | Konsultant | Opcjonalnie |
| `session_channel` | varchar(30) | Czat, telefon, wideo | Mapowanie słownika |
| `started_at` | datetime | Początek sesji | Zachować |
| `ended_at` | datetime null | Koniec sesji | Zachować |
| `resolution_summary` | text null | Wynik wsparcia | Zachować |
| `resolved_without_shipment` | bit | Czy sprawa zakończona zdalnie | Kluczowe dla klasyfikacji |

#### `request_messages`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz wiadomości | Techniczne |
| `service_request_id` | bigint FK -> service_requests.id | Zgłoszenie | Relacja obowiązkowa |
| `author_user_id` | bigint FK -> users.id | Autor wiadomości | Relacja obowiązkowa |
| `message_type` | varchar(30) | `customer`, `internal_note`, `system` | Mapowanie słownika |
| `body` | text | Treść komunikatu | Zachować zgodnie z retencją |
| `created_at` | datetime | Czas wpisu | Zachować |

### 4.4 Tabele logistyczne

#### `shipments`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz przesyłki | Zachować jako `legacy_shipment_id` |
| `service_request_id` | bigint FK -> service_requests.id | Zgłoszenie | Relacja obowiązkowa |
| `shipment_type` | varchar(20) | `inbound`, `return` | Zachować |
| `carrier_code` | varchar(50) | Operator logistyczny | Mapowanie słownika |
| `tracking_number` | varchar(100) | Numer śledzenia | Zachować |
| `label_url` | varchar(500) null | Link do etykiety | Opcjonalnie |
| `current_status` | varchar(50) | Bieżący status przesyłki | Zachować |
| `sent_at` | datetime null | Nadanie | Opcjonalnie |
| `delivered_at` | datetime null | Dostarczenie | Opcjonalnie |
| `last_location_text` | varchar(255) null | Ostatnia lokalizacja | Opcjonalnie |

#### `shipment_tracking_events`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz zdarzenia | Techniczne |
| `shipment_id` | bigint FK -> shipments.id | Przesyłka | Relacja obowiązkowa |
| `carrier_status_code` | varchar(50) | Kod statusu przewoźnika | Zachować |
| `carrier_status_label` | varchar(255) | Opis statusu | Zachować |
| `event_time` | datetime | Czas zdarzenia | Zachować |
| `location_text` | varchar(255) null | Lokalizacja | Opcjonalnie |
| `latitude` | decimal(9,6) null | Szerokość geograficzna | Opcjonalnie |
| `longitude` | decimal(9,6) null | Długość geograficzna | Opcjonalnie |
| `raw_payload_ref` | varchar(255) null | Referencja do danych surowych | Opcjonalnie |

### 4.5 Tabele diagnostyki, wycen i napraw

#### `diagnostics`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz diagnozy | Zachować jako `legacy_diagnostic_id` |
| `service_request_id` | bigint FK -> service_requests.id | Zgłoszenie | Relacja obowiązkowa |
| `engineer_employee_id` | bigint FK -> employee_profiles.id | Inżynier | Opcjonalnie |
| `diagnosis_summary` | text | Wynik diagnozy | Zachować |
| `root_cause` | text null | Przyczyna usterki | Opcjonalnie |
| `recommended_action` | text null | Rekomendowane działanie | Opcjonalnie |
| `diagnosed_at` | datetime | Czas diagnozy | Zachować |

#### `quotes`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz wyceny | Zachować jako `legacy_quote_id` |
| `service_request_id` | bigint FK -> service_requests.id | Zgłoszenie | Relacja obowiązkowa |
| `diagnostic_id` | bigint FK -> diagnostics.id null | Diagnoza | Opcjonalnie |
| `quote_no` | varchar(50) | Numer wyceny | Zachować |
| `parts_cost` | decimal(12,2) | Koszt części | Zachować |
| `labor_cost` | decimal(12,2) | Koszt pracy | Zachować |
| `shipping_cost` | decimal(12,2) | Koszt logistyki | Zachować |
| `total_cost` | decimal(12,2) | Całkowity koszt | Zachować |
| `currency_code` | varchar(3) | Waluta | Zachować |
| `approval_status` | varchar(30) | Status akceptacji | Mapowanie obowiązkowe |
| `issued_at` | datetime | Data wystawienia | Zachować |
| `responded_at` | datetime null | Data decyzji klienta | Opcjonalnie |

#### `quote_items`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz pozycji wyceny | Techniczne |
| `quote_id` | bigint FK -> quotes.id | Wycena | Relacja obowiązkowa |
| `item_type` | varchar(20) | `part`, `labor`, `fee` | Mapowanie słownika |
| `item_name` | varchar(255) | Nazwa pozycji | Zachować |
| `quantity` | decimal(10,2) | Ilość | Zachować |
| `unit_price` | decimal(12,2) | Cena jednostkowa | Zachować |
| `line_total` | decimal(12,2) | Wartość pozycji | Zachować |

#### `repairs`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz naprawy | Zachować jako `legacy_repair_id` |
| `service_request_id` | bigint FK -> service_requests.id | Zgłoszenie | Relacja obowiązkowa |
| `quote_id` | bigint FK -> quotes.id null | Wycena | Opcjonalnie |
| `engineer_employee_id` | bigint FK -> employee_profiles.id null | Wykonawca | Opcjonalnie |
| `repair_status` | varchar(30) | Status naprawy | Mapowanie obowiązkowe |
| `started_at` | datetime null | Start naprawy | Opcjonalnie |
| `completed_at` | datetime null | Koniec naprawy | Opcjonalnie |
| `test_result` | varchar(30) null | Wynik testów | Opcjonalnie |
| `repair_summary` | text null | Podsumowanie prac | Zachować |

#### `repair_parts`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz użycia części | Techniczne |
| `repair_id` | bigint FK -> repairs.id | Naprawa | Relacja obowiązkowa |
| `part_no` | varchar(100) | Numer części | Zachować |
| `part_name` | varchar(255) | Nazwa części | Zachować |
| `quantity` | decimal(10,2) | Ilość | Zachować |
| `warehouse_location` | varchar(100) null | Lokalizacja magazynowa | Opcjonalnie |
| `unit_cost` | decimal(12,2) | Koszt jednostkowy | Zachować |

### 4.6 Tabele płatności i rozliczeń

#### `payments`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz płatności | Zachować jako `legacy_payment_id` |
| `service_request_id` | bigint FK -> service_requests.id | Zgłoszenie | Relacja obowiązkowa |
| `quote_id` | bigint FK -> quotes.id null | Wycena | Opcjonalnie |
| `payment_provider` | varchar(50) | Operator płatności | Mapowanie słownika |
| `payment_reference` | varchar(100) | Referencja transakcji | Zachować |
| `amount` | decimal(12,2) | Kwota | Zachować |
| `currency_code` | varchar(3) | Waluta | Zachować |
| `payment_status` | varchar(30) | Status płatności | Mapowanie obowiązkowe |
| `paid_at` | datetime null | Data opłacenia | Opcjonalnie |
| `created_at` | datetime | Data rejestracji | Zachować |

#### `billing_documents`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz dokumentu | Zachować jako `legacy_billing_document_id` |
| `payment_id` | bigint FK -> payments.id null | Płatność | Opcjonalnie |
| `service_request_id` | bigint FK -> service_requests.id | Zgłoszenie | Relacja obowiązkowa |
| `document_type` | varchar(30) | `invoice`, `proforma`, `correction` | Mapowanie słownika |
| `document_no` | varchar(100) | Numer dokumentu | Zachować |
| `issued_at` | datetime | Data wystawienia | Zachować |
| `gross_amount` | decimal(12,2) | Kwota brutto | Zachować |
| `currency_code` | varchar(3) | Waluta | Zachować |
| `document_url` | varchar(500) null | Odnośnik do pliku | Opcjonalnie |

### 4.7 Tabele załączników i audytu

#### `attachments`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz załącznika | Zachować jako `legacy_attachment_id` |
| `entity_type` | varchar(50) | Typ encji powiązanej | Np. zgłoszenie, wycena, naprawa |
| `entity_id` | bigint | ID encji powiązanej | Wymaga mapowania po migracji |
| `file_name` | varchar(255) | Nazwa pliku | Zachować |
| `file_path` | varchar(500) | Ścieżka lub URI | Wymaga remapowania magazynu plików |
| `mime_type` | varchar(100) | Typ MIME | Zachować |
| `uploaded_by_user_id` | bigint FK -> users.id | Autor dodania | Opcjonalnie |
| `created_at` | datetime | Data dodania | Zachować |

#### `audit_logs`

| Kolumna | Typ | Opis | Uwagi migracyjne |
|---|---|---|---|
| `id` | bigint PK | Klucz logu | Techniczne |
| `entity_type` | varchar(50) | Typ encji | Zachować |
| `entity_id` | bigint | ID encji | Zachować jako referencję legacy |
| `action_code` | varchar(50) | Typ akcji | Zachować |
| `performed_by_user_id` | bigint FK -> users.id null | Autor działania | Opcjonalnie |
| `performed_at` | datetime | Czas akcji | Zachować |
| `old_value_json` | text null | Stan przed zmianą | Opcjonalnie |
| `new_value_json` | text null | Stan po zmianie | Opcjonalnie |

---

## 5. Relacje kluczowe

Najważniejsze zależności, które muszą zostać zachowane podczas migracji:

- `users` 1..1 `customer_profiles` lub `employee_profiles`,
- `users` M..N `roles` przez `user_roles`,
- `customer_profiles` 1..N `devices`,
- `customer_profiles` 1..N `service_requests`,
- `devices` 1..N `service_requests`,
- `service_requests` 1..N `request_status_history`,
- `service_requests` 1..N `request_messages`,
- `service_requests` 0..N `remote_support_sessions`,
- `service_requests` 0..N `shipments`,
- `shipments` 1..N `shipment_tracking_events`,
- `service_requests` 0..1 `diagnostics` lub 1..N, jeżeli wykonawca przyjmie model wielodiagnozowy,
- `service_requests` 0..N `quotes`,
- `quotes` 1..N `quote_items`,
- `service_requests` 0..N `repairs`,
- `repairs` 0..N `repair_parts`,
- `service_requests` 0..N `payments`,
- `service_requests` 0..N `billing_documents`,
- różne encje 0..N `attachments`,
- różne encje 0..N `audit_logs`.

---

## 6. Referencyjne wolumeny danych do estymacji migracji

Na potrzeby wyceny i zaprojektowania procesu migracji należy przyjąć następujące orientacyjne wolumeny danych historycznych. Wartości zostały oszacowane na podstawie agregatów miesięcznych i rocznych z wewnętrznych raportów operacyjnych, a nie na podstawie pełnego zrzutu produkcyjnej bazy danych:

| Tabela / obszar | Szacowany wolumen |
|---|---|
| `users` | 125 000 |
| `customer_profiles` | 104 000 |
| `employee_profiles` | 450 |
| `devices` | 168 000 |
| `service_requests` | 1 050 000 |
| `request_status_history` | 9 800 000 |
| `request_messages` | 3 600 000 |
| `remote_support_sessions` | 270 000 |
| `shipments` | 1 120 000 |
| `shipment_tracking_events` | 6 400 000 |
| `diagnostics` | 590 000 |
| `quotes` | 470 000 |
| `quote_items` | 1 450 000 |
| `repairs` | 430 000 |
| `repair_parts` | 980 000 |
| `payments` | 410 000 |
| `billing_documents` | 430 000 |
| `attachments` | 1 900 000 rekordów metadanych |
| `audit_logs` | 21 000 000 |

Wolumeny mają charakter referencyjny i służą do oszacowania podejścia, wydajności migracji, narzędzi ETL oraz czasu potrzebnego na migrację próbną i końcową.

---

## 7. Priorytety migracyjne

### 7.1 Zakres obowiązkowy przed startem produkcyjnym

Przed uruchomieniem nowego systemu należy zmigrować co najmniej:

- aktywnych klientów,
- aktywne konta pracowników,
- wszystkie otwarte zgłoszenia,
- urządzenia powiązane z otwartymi zgłoszeniami,
- historię statusów dla zgłoszeń aktywnych,
- aktywne przesyłki i ich historię,
- aktualne diagnozy, wyceny i decyzje klienta,
- niezbędne płatności i dokumenty dla spraw aktywnych,
- załączniki wymagane operacyjnie dla spraw aktywnych.

### 7.2 Zakres historyczny po starcie lub w migracji warstwowej

Po starcie produkcyjnym dopuszcza się migrację etapową:

- zamkniętych zgłoszeń z ostatnich 24 miesięcy,
- pełnej historii płatności i dokumentów z ostatnich 36 miesięcy,
- starszych załączników i logów audytowych,
- pozostałych danych archiwalnych wykorzystywanych głównie do raportowania.

---

## 8. Zasady mapowania i jakości danych

Wykonawca powinien założyć występowanie następujących problemów jakościowych po stronie danych legacy:

- brak części pól opcjonalnych w rekordach historycznych,
- niespójne nazwy statusów pomiędzy starszymi i nowszymi rekordami,
- duplikaty klientów wykrywane po e-mailu, telefonie lub danych firmy,
- duplikaty urządzeń wykrywane po numerze seryjnym,
- pola tekstowe zawierające wartości słownikowe zamiast kluczy referencyjnych,
- niepełne dane lokalizacyjne przesyłek,
- brak części załączników fizycznych mimo obecności metadanych.

Minimalne zasady migracyjne:

1. Każdy rekord biznesowy przeniesiony do systemu docelowego powinien zachować identyfikator legacy w osobnym polu referencyjnym.
2. Numery biznesowe, takie jak `request_no`, `quote_no`, `document_no` i `tracking_number`, powinny zostać zachowane bez zmian, o ile nie kolidują z modelem docelowym.
3. Relacje klient-urządzenie-zgłoszenie-przesyłka-wycena-płatność muszą zostać odtworzone deterministycznie.
4. Rekordy niekompletne powinny zostać oznaczone flagą migracyjną, a nie pominięte bez raportu.
5. Wykonawca powinien przygotować raport rozbieżności po każdej migracji próbnej.

---

## 9. Oczekiwany mechanizm migracji

W ofercie wykonawca powinien opisać docelowy mechanizm migracyjny obejmujący co najmniej:

- ekstrakcję danych z systemu źródłowego do formatu pośredniego,
- warstwę mapowania i czyszczenia danych,
- walidację referencyjną oraz walidację integralności,
- obsługę migracji przyrostowej dla danych zmienianych w okresie przejściowym,
- repozytorium mapowań identyfikatorów legacy-do-new,
- migrację załączników i dokumentów binarnych,
- zestaw raportów kontrolnych po migracji,
- procedurę cutover na środowisko produkcyjne,
- plan rollback lub plan bezpiecznego zatrzymania procesu migracji w razie błędu krytycznego.

---

## 10. Oczekiwane artefakty od wykonawcy

Wykonawca powinien dostarczyć w ramach projektu co najmniej:

- dokument mapowania danych źródłowych do modelu docelowego,
- listę reguł transformacji i walidacji,
- plan migracji próbnej,
- raport wyników migracji próbnej,
- plan migracji końcowej,
- raport zgodności danych po migracji końcowej,
- instrukcję operacyjną uruchomienia mechanizmu migracyjnego.
