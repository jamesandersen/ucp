<!--
   Copyright 2026 UCP Authors

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

# Buyer Consent Extension

## Overview

The Buyer Consent extension enables platforms to transmit buyer consent choices
to businesses regarding data usage and communication preferences. It allows
buyers to communicate their consent status for various categories, such as
analytics, marketing, and data sales, helping businesses comply with privacy
regulations like CCPA and GDPR.

When this extension is supported, the `buyer` object in checkout is extended
with a `consent` field containing consent states, and the checkout response
may include `marketing_consent_options` declaring which marketing channels the
business offers for opt-in.

This extension can be included in `create_checkout`, `update_checkout`, and
`complete_checkout` operations.

## Discovery

Businesses advertise consent support in their profile:

```json
{
  "capabilities": {
    "dev.ucp.shopping.buyer_consent": [
      {
        "version": "{{ ucp_version }}",
        "extends": "dev.ucp.shopping.checkout"
      }
    ]
  }
}
```

## Schema Composition

The consent extension extends the **buyer object** and the **checkout object**:

- **Base schema extended**: `checkout` via `buyer` object and top-level
    `marketing_consent_options`
- **Paths**:
    - `checkout.buyer.consent` — buyer consent states
    - `checkout.marketing_consent_options` — business-declared marketing
        channels (response only)
- **Schema reference**: `buyer_consent.json`

## Schema Definition

### Consent Object

{{ extension_schema_fields('buyer_consent.json#/$defs/consent', 'buyer-consent') }}

### Marketing Consent Option

{{ extension_schema_fields('buyer_consent.json#/$defs/marketing_consent_option', 'buyer-consent') }}

### Marketing Consent

{{ extension_schema_fields('buyer_consent.json#/$defs/marketing_consent', 'buyer-consent') }}

## Usage

### Data Processing Consent

The platform includes data processing consent within the `buyer` object in
checkout operations:

#### Example: Create Checkout with Data Processing Consent

```json
POST /checkouts

{
  "line_items": [
    {
      "item": {
        "id": "prod_123",
        "title": "Blue T-Shirt",
        "price": 1999
      },
      "id": "li_1",
      "quantity": 1
    }
  ],
  "buyer": {
    "email": "jane.doe@example.com",
    "first_name": "Jane",
    "last_name": "Doe",
    "consent": {
      "analytics": true,
      "preferences": true,
      "sale_of_data": false
    }
  }
}
```

### Channel-Specific Marketing Consent

Marketing consent uses a two-part design: the business declares which marketing
channels it offers, and the platform passes the buyer's consent decisions.

#### Step 1: Business Declares Marketing Consent Options (Response)

When the business supports marketing opt-in, it includes
`marketing_consent_options` in the checkout response. Each option specifies the
channel type, a description of what the buyer is consenting to, and a link to
the business's privacy policy.

```json
{
  "id": "checkout_456",
  "status": "ready_for_complete",
  "currency": "USD",
  "marketing_consent_options": [
    {
      "type": "email",
      "description": "Promotional emails, product launches, and exclusive offers",
      "privacy_policy_url": "https://example.com/privacy"
    },
    {
      "type": "sms",
      "description": "Order updates and flash sale alerts via text message",
      "privacy_policy_url": "https://example.com/privacy"
    }
  ],
  "buyer": {
    "email": "jane.doe@example.com",
    "first_name": "Jane",
    "last_name": "Doe"
  },
  "line_items": ["..."],
  "totals": ["..."],
  "links": [
    {
      "type": "privacy_policy",
      "url": "https://example.com/privacy"
    }
  ]
}
```

#### Step 2: Platform Passes Buyer Consent Decisions (Request)

When the checkout response includes `marketing_consent_options`, the platform
SHOULD display these to the buyer and include the buyer's decisions in
`buyer.consent.marketing_consents` on the complete checkout request.

```json
POST /checkouts/checkout_456/complete

{
  "buyer": {
    "consent": {
      "marketing_consents": [
        {
          "type": "email",
          "opted_in": true
        },
        {
          "type": "sms",
          "opted_in": false
        }
      ]
    }
  },
  "payment": {
    "instruments": ["..."]
  }
}
```

### Backward Compatibility

The `marketing` boolean field is deprecated but remains supported. Platforms
and businesses that have not adopted channel-specific consent can continue
using `marketing: true` or `marketing: false` as a blanket signal.

When both `marketing` and `marketing_consents` are present, businesses MUST
prefer `marketing_consents`. When only `marketing: true` is present without
`marketing_consents`, businesses MAY treat it as consent for all channels
they offer.

## Platform Behavior

When a checkout response includes `marketing_consent_options`, the platform
SHOULD:

1. Compose display text using the business name and the option's `description`
   (e.g., "Receive promotional emails, product launches, and exclusive offers
   from ExampleStore")
2. Render each option as a checkbox, always **unchecked by default**
3. Link to the business's `privacy_policy_url` for the buyer to review
4. Include the buyer's decisions in `buyer.consent.marketing_consents` on the
   complete checkout request
5. If the buyer does not interact with the consent UI, the platform SHOULD
   send `opted_in: false` for each channel or omit `marketing_consents`
   entirely (both treated as no consent)

## Contact Resolution

Marketing consent applies to the contact information available on the buyer
object. For `email` consent, the business uses `buyer.email`. For `sms` or
`whatsapp` consent, the business uses `buyer.phone_number`.

If `marketing_consents` includes a channel type but the buyer object does not
contain the corresponding contact information (e.g., `sms` consent without
`buyer.phone_number`), the business MUST ignore the consent for that channel.

## Business Behavior

1. Businesses MUST NOT treat omission of `marketing_consents` as consent.
   Absence means no consent was given.
2. Businesses MUST respect the buyer's `opted_in` value and MUST NOT send
   marketing communications to buyers who did not opt in.
3. Businesses SHOULD only include `marketing_consent_options` for channels
   they actively support.

## Security & Privacy Considerations

1. **Consent is declarative** — The protocol communicates consent, it does not
   enforce it
2. **Legal compliance** remains the business's responsibility
3. **Platforms should not** assume consent without explicit user action
4. **Default behavior** when consent is not provided is business-specific
5. **Unchecked by default** — Marketing opt-in checkboxes MUST default to
   unchecked to comply with GDPR requirements for explicit, informed consent
6. **Channel scoping** — Consent is scoped to specific channels. Consent for
   one channel (e.g., email) does not imply consent for another (e.g., SMS)
