# Home Currency Architecture

Home Currency uses a lightweight mobile-and-cloud architecture designed around three goals:

- keep currency conversion fast on the device
- avoid tightly coupling the mobile app to a third-party exchange-rate provider
- preserve useful functionality when network connectivity is unavailable

---

## Overview

```text
Exchange Rate Provider
        |
        v
    AWS Lambda
        |
        v
 Normalize / Validate
        |
        v
    Amazon S3
        |
        v
       CDN
        |
        v
 Flutter Mobile App
        |
        +--> Local Rate Cache
        |
        +--> Currency Conversion
        |
        +--> Calculator Workflow
```

The mobile application does not call the external exchange-rate provider for every conversion.

Instead, exchange-rate data is retrieved on a scheduled basis, normalized by the backend, published as a stable JSON resource, and then consumed by the app.

---

## Cloud Data Flow

### 1. Scheduled Refresh

Amazon EventBridge triggers the exchange-rate refresh process on a daily schedule.

This removes the need for a continuously running backend service.

### 2. Rate Retrieval

AWS Lambda retrieves the latest exchange-rate data from the configured provider.

The backend is responsible for interacting with the provider rather than exposing that dependency directly to the mobile application.

### 3. Normalization

Provider data is transformed into the Home Currency rate format.

This creates a consistent contract between the cloud layer and the mobile client.

The rate payload includes a schema version so future data-format changes can be handled intentionally.

Current schema:

```text
schemaVersion: 1
```

### 4. Publication

The normalized file is written to Amazon S3 as:

```text
rates-latest.json
```

The file is then made available to the mobile application through a CDN endpoint.

---

## Mobile Data Flow

When the application launches or refreshes its rate data:

1. The app attempts to retrieve the latest rate file.
2. The payload is validated.
3. Valid rates are stored locally.
4. The latest successful rate set becomes the active conversion data.
5. The UI displays information about when rates were last downloaded.

Currency calculations then occur locally on the device.

This means normal conversions do not require a network request.

---

## Offline Behavior

Travel environments often include:

- airport Wi-Fi
- roaming transitions
- limited mobile data
- temporary loss of connectivity
- locations with unreliable service

Home Currency therefore retains the most recent valid exchange-rate data locally.

If a network request fails, the app can continue using the cached rate set.

This allows the core conversion workflow to remain available offline.

The interface also communicates the age of the stored rate data so the user has context about freshness.

---

## Why the App Does Not Query the Provider Directly

Calling an exchange-rate API directly from the mobile client would be simpler initially, but it would introduce several disadvantages.

### Provider Coupling

The mobile app would depend on the exact API structure of the external provider.

Any provider change could require a mobile application update.

### Credential Exposure

Providers that require API credentials are more difficult to protect when credentials are embedded in a distributed mobile application.

### Inconsistent Data

Different providers may use different response structures, naming conventions, or base currencies.

Normalizing data in the backend keeps those differences away from the client.

### Request Volume

A centrally refreshed rate file reduces unnecessary third-party API traffic.

---

## Provider Fallback Strategy

The backend architecture is designed so the external rate source can be replaced or supplemented without changing the mobile application's data contract.

Conceptually:

```text
Primary Rate Provider
        |
        | failure
        v
Fallback Rate Provider
        |
        v
Normalization Layer
        |
        v
rates-latest.json
```

The mobile application continues consuming the same Home Currency schema regardless of which provider supplied the source data.

This keeps provider-specific logic out of the UI and conversion workflow.

---

## Calculator Integration

Each currency field includes access to a calculator.

The calculator is part of the conversion workflow rather than a separate utility.

Typical use cases include:

- applying a discount
- calculating tax
- multiplying a unit price
- splitting a bill
- combining purchases
- calculating a tip

The calculator's **Use Result** action sends the calculated value directly back into the selected currency field.

The conversion result is then recalculated automatically.

This removes the need to copy a value from another calculator application and paste it into Home Currency.

---

## Application Responsibilities

### Flutter Mobile App

Responsible for:

- user interface
- currency selection
- local calculations
- calculator workflow
- local rate caching
- rate freshness display
- advertisement display
- remove-ads feature gating

### AWS Lambda

Responsible for:

- retrieving external rate data
- validating provider responses
- normalizing the data
- generating the application rate payload

### Amazon EventBridge

Responsible for:

- triggering scheduled exchange-rate refreshes

### Amazon S3

Responsible for:

- storing the normalized rate file

### CDN Layer

Responsible for:

- delivering the rate file efficiently to mobile clients

---

## Failure Scenarios

### External Provider Unavailable

The scheduled refresh may fail, but the previously published rate file remains available.

The mobile application can continue using the most recently retrieved rate data.

### Mobile Device Offline

The app continues using its local cached rates.

### Rate Download Fails

The last valid local rate set remains active rather than being replaced with invalid or incomplete data.

### Provider Changes Format

The normalization layer isolates the mobile application from most provider-specific changes.

---

## Design Principle

The central design principle is:

> The mobile app should understand the Home Currency data contract, not the external provider.

This separation keeps the mobile application simpler and makes the exchange-rate source easier to change over time.

---

## Future Architecture Work

Planned architecture improvements include:

- production-ready fallback-provider handling
- additional payload validation
- improved network error reporting
- cache-expiration rules
- telemetry and diagnostics
- production in-app purchase integration
- OCR-based price capture
- broader automated testing

Phase 2 will introduce OCR-based price capture, allowing scanned prices to enter the existing conversion workflow.

---

## Summary

Home Currency uses a deliberately small architecture:

```text
Scheduled Cloud Refresh
        +
Stable JSON Contract
        +
Local Mobile Cache
        =
Fast, resilient currency conversion
```

The approach is intended to keep the application lightweight while providing enough separation between the mobile client, cloud infrastructure, and external data providers to support future growth.
