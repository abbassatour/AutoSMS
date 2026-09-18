
# Auto SMS — Enterprise CRM & Automated SMS Dispatcher

[![Flutter](https://img.shields.io/badge/Flutter-3.41.0-02569B?logo=flutter)](https://flutter.dev)
[![Dart](https://img.shields.io/badge/Dart-3.11.0-0175C2?logo=dart)](https://dart.dev)
[![Architecture: Very Good](<https://img.shields.io/badge/Architecture-Very%20Good%20CLI-B22C89>)](https://verygood.ventures)
[![State Management: Bloc](<https://img.shields.io/badge/State%20Management-Bloc%20%2F%20Cubit-blue>)](https://bloclibrary.dev)
[![Database: Drift SQLite](<https://img.shields.io/badge/Local%20DB-Drift%20SQLite-00599C>)](https://drift.simonbinder.eu)
[![Cloud Backend: Supabase](https://img.shields.io/badge/Backend-Supabase-3ECF8E?logo=supabase)](https://supabase.com)
[![Push Service: Firebase FCM](<https://img.shields.io/badge/Push-Firebase%20Cloud%20Messaging-FFCA28?logo=firebase>)](https://firebase.google.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

An enterprise-grade, offline-first mobile CRM and automated SMS dispatching platform built with Flutter. The system allows businesses to manage contacts and customer segments, assign scheduled SMS campaigns to specific connected Android devices, and reliably dispatch automated messages in the background using Firebase Cloud Messaging (FCM) silent triggers orchestrated by Supabase Edge Functions.

---

## 📑 Table of Contents

- [System Architecture &amp; Workflow](#-system-architecture--workflow)
- [Key Features](#-key-features)
- [Monorepo Package Structure](#-monorepo-package-structure)
- [Prerequisites](#-prerequisites)
- [Environment Configuration](#-environment-configuration)
- [Supabase &amp; Cloud Backend Setup](#-supabase--cloud-backend-setup)
- [Firebase Setup](#-firebase-setup)
- [Installation &amp; Build Guide](#-installation--build-guide)
- [Flavors &amp; Execution](#-flavors--execution)
- [Localization (i18n)](#-localization-i18n)
- [Testing &amp; Quality Assurance](#-testing--quality-assurance)

---

## 🔄 System Architecture & Workflow

The platform operates on an offline-first, distributed execution model:

```
[ Supabase Edge Cron ] ──> Checks active schedules every minute
         │
         ▼
[ Firebase Cloud Messaging ] ──> Sends High-Priority Silent Data Message
         │
         ▼
[ Android Background Worker ] ──> Wakes phone isolate via `@pragma('vm:entry-point')`
         │
         ├──> Loads target group contacts from local Drift SQLite
         ├──> Sends native SMS messages via Telephony engine (with automatic retries)
         ├──> Records dispatch logs locally in SQLite database
         └──> Syncs logs and metadata back to Supabase Cloud
```

1. **Scheduling Engine:** Campaigns are configured locally and synced to Supabase. A serverless Edge Function (`auto_sms_cron`) executes on a schedule, evaluates schedules (configured for Damascus UTC+3 time), and detects campaigns that must run at the current minute.
2. **Device Awakening:** The Edge Function dispatches an FCM silent data message directly to the specific Android device registered to execute the campaign.
3. **Background Dispatcher:** The device receives the payload in `_firebaseMessagingBackgroundHandler`, initializes local storage and repository instances, queries the target group's contacts, and sends the SMS messages natively using Android telephony APIs.
4. **Resilient Synchronization:** Dispatch status logs (`sent_auto_fcm` or `failed_auto_fcm`) are saved locally to SQLite immediately, ensuring audit trails are never lost, and then pushed to Supabase when network connectivity is available.

---

## 🚀 Key Features

- **Offline-First CRM:**
  - Fast local storage using **Drift (SQLite)**.
  - Soft-deletion support across groups, contacts, and schedules to prevent data desynchronization between clients.
  - Smart bi-directional synchronization with Supabase (`downloadIfCloudIsNewer` and `syncAllToCloud`).
- **Contact & Group Management:**
  - Import device contacts directly into the CRM (`flutter_contacts`).
  - Search, filter by group, and execute bulk group assignments.
  - Quick action integrations: Direct Phone Call, WhatsApp chat launch, and native SMS composers (`url_launcher`).
- **Automated Campaign Scheduling:**
  - Schedule recurring monthly campaigns by specifying day of the month (1–31), hour, and minute.
  - Bind specific campaigns to dedicated sender devices.
  - Toggle campaign states on/off with instantaneous synchronization.
- **Hardware-Linked Device Management:**
  - Register sending phones with persistent hardware fingerprints (`AndroidId`) and unique names.
  - Dynamic device unlinking and remote FCM token rotation.
- **Bilingual Internationalization:**
  - Native localization for English (`en`) and Arabic (`ar`) with complete Right-to-Left (RTL) layout support.

---

## 📦 Monorepo Package Structure

This project follows a clean, layered monorepo architecture:

```
auto_sms/
├── lib/                             # Core Flutter Application
│   ├── app/                         # App initialization, AuthGate, WorkspaceInitializer
│   ├── campaigns/                   # Campaigns and groups UI & Cubit
│   ├── contacts/                    # Contacts CRM UI & Cubit
│   ├── dashboard/                   # Monitoring dashboard, device linker & Cubit
│   ├── home/                        # Bottom navigation view & Cubit
│   ├── login/                       # Supabase Authentication UI & Cubit
│   ├── l10n/                        # ARB localization definitions and generated files
│   ├── bootstrap.dart               # Global error boundaries and BlocObserver
│   ├── firebase_options.dart        # Platform-specific Firebase credentials
│   ├── main_development.dart       # Development entry point
│   ├── main_production.dart        # Production entry point
│   └── main_staging.dart           # Staging entry point
│
├── packages/
│   ├── local_storage_api/           # Drift SQLite database schema, tables & queries
│   ├── cloud_storage_api/           # Supabase client wrapper, Auth & cloud tables
│   └── crm_repository/              # Business repository orchestrating Local + Cloud
│
└── supabase/
    ├── config.toml                  # Supabase local development configuration
    └── functions/
        └── auto_sms_cron/           # Deno Edge Function for automated cron evaluation
```

---

## 🛠 Prerequisites

Ensure you have the following installed on your development workstation:

- **Flutter SDK:** `>= 3.41.0`
- **Dart SDK:** `>= 3.11.0`
- **Android SDK:** API Level 34+ (SMS and contact permissions required for testing)
- **Supabase CLI:** For local development or Edge Function deployment
- **Deno:** Required for running and debugging Supabase Edge Functions locally
- **Firebase Project:** Configured with Cloud Messaging enabled

---

## 🔐 Environment Configuration

This project eliminates hardcoded credentials by injecting secrets at compile time via `String.fromEnvironment` through `lib/app/config/env_config.dart`.

Create a `config.json` file in the root directory (this file is git-ignored):

```json
{
  "SUPABASE_URL": "https://your-project-id.supabase.co",
  "SUPABASE_ANON_KEY": "your-supabase-anon-key",
  "FIREBASE_PROJECT_ID": "your-firebase-project-id",
  "FIREBASE_STORAGE_BUCKET": "your-firebase-project-id.appspot.com",
  "FIREBASE_MESSAGING_SENDER_ID": "your-sender-id",
  "FIREBASE_ANDROID_API_KEY": "your-android-api-key",
  "FIREBASE_ANDROID_APP_ID": "1:000000000000:android:0000000000000000000000",
  "FIREBASE_WINDOWS_API_KEY": "your-windows-api-key",
  "FIREBASE_WINDOWS_APP_ID": "1:000000000000:web:0000000000000000000000",
  "FIREBASE_AUTH_DOMAIN": "your-firebase-project-id.firebaseapp.com",
  "FIREBASE_MEASUREMENT_ID": "G-XXXXXXXXXX"
}
```

---

## 🗄 Supabase & Cloud Backend Setup

### 1. Database Schema

Execute the following SQL queries in the Supabase SQL Editor to configure your remote PostgreSQL database:

```sql
-- Enable UUID extension
create extension if not exists "uuid-ossp";

-- Groups Table
create table public.groups (
    id text primary key,
    user_id uuid references auth.users(id) on delete cascade not null,
    name text not null,
    is_deleted boolean default false not null,
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Contacts Table
create table public.contacts (
    id text primary key,
    user_id uuid references auth.users(id) on delete cascade not null,
    name text not null,
    phone text not null,
    group_id text references public.groups(id) on delete set null,
    is_deleted boolean default false not null,
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Schedules Table
create table public.schedules (
    id text primary key,
    user_id uuid references auth.users(id) on delete cascade not null,
    group_id text references public.groups(id) on delete cascade not null,
    message text not null,
    send_day integer not null,
    send_hour integer default 9 not null,
    send_minute integer default 0 not null,
    target_device_id text,
    last_sent_date timestamp with time zone,
    is_active boolean default true not null,
    is_deleted boolean default false not null,
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Message Logs Table
create table public.messages (
    id text primary key,
    user_id uuid references auth.users(id) on delete cascade not null,
    phone text not null,
    body text not null,
    type text not null,
    message_date timestamp with time zone not null
);

-- Device Registration & FCM Tokens
create table public.user_tokens (
    device_id uuid default gen_random_uuid() primary key,
    user_id uuid references auth.users(id) on delete cascade not null,
    device_name text not null,
    fcm_token text not null,
    hardware_id text not null,
    updated_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Sync Metadata Tracking
create table public.sync_metadata (
    user_id uuid references auth.users(id) on delete cascade primary key,
    last_updated_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Row Level Security (RLS)
alter table public.groups enable row level security;
alter table public.contacts enable row level security;
alter table public.schedules enable row level security;
alter table public.messages enable row level security;
alter table public.user_tokens enable row level security;
alter table public.sync_metadata enable row level security;

-- Policies for Authenticated Users
create policy "Users can manage their own groups" on public.groups for all using (auth.uid() = user_id);
create policy "Users can manage their own contacts" on public.contacts for all using (auth.uid() = user_id);
create policy "Users can manage their own schedules" on public.schedules for all using (auth.uid() = user_id);
create policy "Users can manage their own messages" on public.messages for all using (auth.uid() = user_id);
create policy "Users can manage their own devices" on public.user_tokens for all using (auth.uid() = user_id);
create policy "Users can manage their own metadata" on public.sync_metadata for all using (auth.uid() = user_id);
```

### 2. Deploying the Edge Function

The Edge Function handles cron evaluation and triggers silent FCM messages:

```bash
# Set your Firebase Service Account JSON inside Supabase Secrets
supabase secrets set FIREBASE_SERVICE_ACCOUNT='{"type":"service_account",...}'

# Deploy the function
supabase functions deploy auto_sms_cron --no-verify-jwt
```

You can schedule this function via `pg_cron` in Supabase to run every minute:

```sql
select cron.schedule(
  'auto-sms-cron-job',
  '* * * * *',
  $$
  select
    net.http_post(
      url:='https://your-project-id.supabase.co/functions/v1/auto_sms_cron',
      headers:='{"Content-Type": "application/json", "Authorization": "Bearer YOUR_SERVICE_ROLE_KEY"}'::jsonb
    ) as request_id;
  $$
);
```

---

## 🔥 Firebase Setup

1. Create a Firebase project at [Firebase Console](https://console.firebase.google.com/).
2. Add an Android app with the package name defined in your `android/app/build.gradle`.
3. Download `google-services.json` and place it in `android/app/`.
4. Generate a private key for your service account from **Project Settings > Service Accounts** and inject it into the Supabase Edge Function environment secret as described above.

---

## ⚡ Installation & Build Guide

### 1. Fetch Dependencies

Run pub get across all packages in the repository:

```bash
flutter pub get
(cd packages/local_storage_api && flutter pub get)
(cd packages/cloud_storage_api && flutter pub get)
(cd packages/crm_repository && flutter pub get)
```

### 2. Run Code Generation

Generate Drift SQLite database code and localization files:

```bash
# Generate Drift Database Models
(cd packages/local_storage_api && dart run build_runner build --delete-conflicting-outputs)

# Generate Localization Classes
flutter gen-l10n
```

---

## 📱 Flavors & Execution

The project supports three flavors: `development`, `staging`, and `production`.

### Running in Development

```bash
flutter run \
  --flavor development \
  --target lib/main_development.dart \
  --dart-define-from-file=config.json
```

### Running in Production

```bash
flutter run \
  --flavor production \
  --target lib/main_production.dart \
  --dart-define-from-file=config.json
```

### Building Android Release APK

```bash
flutter build apk \
  --flavor production \
  --target lib/main_production.dart \
  --dart-define-from-file=config.json
```

### Building Android App Bundle (AAB)

```bash
flutter build appbundle \
  --flavor production \
  --target lib/main_production.dart \
  --dart-define-from-file=config.json
```

---

## 🌐 Localization (i18n)

The application manages internationalization through Flutter's synthetic `AppLocalizations` package:

- Translation source files: `lib/l10n/arb/app_en.arb` and `lib/l10n/arb/app_ar.arb`.
- Configuration: `l10n.yaml`.
- Generated code output: `lib/l10n/gen/`.

Whenever you add or modify string keys, recompile translations:

```bash
flutter gen-l10n
```

Access localized strings within widgets using the context extension:

```dart
final l10n = context.l10n;
Text(l10n.dashboardTitle);
```

---

## 🧪 Testing & Quality Assurance

### Run Tests

```bash
# Run unit and widget tests with randomized seed
flutter test --test-randomize-ordering-seed random

# Run package-specific tests
(cd packages/crm_repository && flutter test)
(cd packages/local_storage_api && flutter test)
(cd packages/cloud_storage_api && flutter test)
```

### Code Formatting and Static Analysis

The codebase adheres to `very_good_analysis` and `bloc_lint`:

```bash
# Analyze main project
flutter analyze

# Lint Blocs & Cubits
dart run bloc_tools:bloc lint .
```

---

## 🔒 Android Permissions Note

Because this app acts as an enterprise SMS dispatcher, the following permissions are configured in `AndroidManifest.xml` and requested at runtime:

- `android.permission.SEND_SMS`: Required to dispatch campaign SMS messages programmatically.
- `android.permission.READ_PHONE_STATE`: Required by the `telephony` plugin to access SIM network capabilities.
- `android.permission.READ_CONTACTS`: Required to sync phone address books into the CRM.
- `android.permission.POST_NOTIFICATIONS`: Required on Android 13+ (API 33+) for persistent background status indications.
- `android.permission.WAKE_LOCK`: Ensures the background isolate can finish dispatching campaigns before entering deep sleep.
