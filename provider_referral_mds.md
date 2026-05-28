# Provider Referral MDS Receive API Specification v1.0

This document describes the minimum data set (MDS) that Digital Weight Management Programme (DWMP) will send to provider
systems when a service user referral is submitted to a provider.

It is intended for providers who are building an API endpoint that can receive referral
data from DWMP. In this integration, DWMP is the API client/sender and the provider
system is the API server/receiver.

The payload represents a single service user referral. The `providerUbrn` field is the
unique referral identifier and should be used by provider systems when reconciling
received submissions with DWMP.

## Endpoint

Providers should expose an HTTPS only `POST` endpoint capable of receiving the referral MDS as a
JSON request body. The exact base URL will be agreed with each provider during onboarding.
The following path is the recommended resource shape:

```http
POST /api/referral
Content-Type: application/json
```

For an existing provider API, DWMP can implement the existing authentication method. For new API builds; OAuth 2.0 Bearer Token authentication with JWTs (JSON Web Tokens) would be the preferred method.

Provider APIs should return a standard HTTP status code that clearly communicates the outcome of the submission:

| Status Code | Meaning |
| --- | --- |
| `201 Created` | The provider accepted the referral and created a provider-side referral resource. |
| `400 Bad Request` | The payload is invalid and should be corrected before re-submission. |
| `401 Unauthorized` | DWMP is not authenticated to submit the referral. |
| `403 Forbidden` | DWMP is not authorised to submit the referral. |
| `409 Conflict` | A referral with the same `providerUbrn` has already been created. |
| `5xx` | The provider API could not process the request because of a server-side problem. |

### Idempotency
The provider's API should respond in a timely manner as the DWMP request will timeout after 30 seconds. Should a timeout occur, DWMP will resend the request. If the provider's API has idempotency support then the `idempotencyKey` field in the payload can be used to identify a unique request, subsequent requests with the same `idempotencyKey` should return the same response as the original request. If the provider's API does not have idempotency support then, following a timeout where the original request creates the resource, a `409 Conflict` should be returned whose `receivedAtUtc` field will be used by DWMP to determine if the `409 Conflict` is within the timeout retry period and if it is assume the referral has been successfully created by a previous timed out request.

### Rate Limits
In normal operation requests to the provider's API will be made in real time as soon as the service user confirms their provider selection. In downtime recovery situations where a backlog of referrals has built up and communication has been restored, they will be sent sequentially one after the other until the backlog is cleared. They will be sent as quickly as processable by the provider's API, so rate limits should be set accordingly.

## Payload Fields

All fields will always be sent in the request.

| Field | Data Type | Max Length | Format / Allowed Values | Description |
| --- | --- | --- | --- | --- |
| `idempotencyKey` | `string` | 36 | `GUID` | Identifies a unique request, consistent between retries. |
| `providerUbrn` | `string` | 12 | `^\d{12}$` | Unique booking reference number. This is the referral's unique key and is used for all submissions from DWMP to the provider. |
| `dateOfReferral` | `string` | 10 | `yyyy-MM-dd` | The date the referral was received by DWMP. |
| `providerSelectedDate` | `string` | 10 | `yyyy-MM-dd` | The date when the service user selected the provider. |
| `referralSource` | `string` | 50 | `GP`, `NHSStaff`, `Pharmacy`, `MSK`, `ElectiveCare` | The source pathway of the referral. |
| `address1` | `string` | 200 |  | Service user's home address first line. |
| `address2` | `nullable string` | 200 |  | Service user's home address second line or null if not present. |
| `address3` | `nullable string` | 200 |  | Service user's home address third line or null if not present. |
| `age` | `integer` |  | `>= 18`, `<= 110` | Service user's age at `dateOfReferral`. |
| `bmi` | `decimal(18,2)` |  | `>= 27.5`, `<= 90` | Service user's BMI recorded on `bmiDate`. |
| `bmiDate` | `string` | 10 | `yyyy-MM-dd` | The date when the service user's BMI was recorded. |
| `email` | `string` | 200 | RFC 5322 compliant | Service user's email address. |
| `ethnicity` | `string` | 5 | `White`, `Black`, `Asian`, `Mixed`, `Other` | Service user's triaged group ethnicity. |
| `familyName` | `string` | 200 |  | Service user's family name. |
| `givenName` | `string` | 200 |  | Service user's given name. |
| `gpPracticeOdsCode` | `string` | 6 | `^[A-Z][A-Z0-9]{2}\d{3}$` | Service user's GP practice ODS code, or one of the special values listed below. |
| `height` | `decimal(18,2)` |  | `>= 50`, `<= 250` | Service user's height in centimetres. |
| `mobile` | `nullable string` | 16 | `^\+[1-9]\d{1,14}$` | Service user's mobile telephone number in E.164 format. `telephone` or `mobile` or both will be populated. |
| `postcode` | `string` | 10 | BS 7666 compliant | Service user's home address postcode. |
| `sexAtBirth` | `string` | 20 | `NotKnown`, `Male`, `Female`, `NotSpecified` | Service user's sex at birth. |
| `telephone` | `nullable string` | 16 | `^\+[1-9]\d{1,14}$` | Service user's landline telephone number in E.164 format. `telephone` or `mobile` or both will be populated. |
| `triagedLevel` | `integer` |  | `1`, `2`, `3` | Service user's triaged level. |
| `hasArthritisOfHip` | `nullable boolean` |  |  | True if the referring clinician has determined the service user has arthritis of a hip. Null if unknown or the service user does not want to disclose. |
| `hasArthritisOfKnee` | `nullable boolean` |  |  | True if the referring clinician has determined the service user has arthritis of a knee. Null if unknown or the service user does not want to disclose. |
| `hasDiabetesType1` | `nullable boolean` |  |  | True if the referring clinician has determined the service user has diabetes type 1. Null if unknown or the service user does not want to disclose. |
| `hasDiabetesType2` | `nullable boolean` |  |  | True if the referring clinician has determined the service user has diabetes type 2. Null if unknown or the service user does not want to disclose. |
| `hasHypertension` | `nullable boolean` |  |  | True if the referring clinician has determined the service user has hypertension. Null if unknown or the service user does not want to disclose. |
| `hasLearningDisability` | `nullable boolean` |  |  | True if the referring clinician has determined the service user has a learning disability. Null if unknown or the service user does not want to disclose. |
| `hasPhysicalDisability` | `nullable boolean` |  |  | True if the referring clinician has determined the service user has a physical disability. Null if unknown or the service user does not want to disclose. |
| `hasRegisteredSeriousMentalIllness` | `nullable boolean` |  |  | True if the referring clinician has determined the service user has a registered serious mental illness. Null if unknown or the service user does not want to disclose. |
| `isVulnerable` | `nullable boolean` |  |  | True if the referring clinician has determined the service user is vulnerable. Null if unknown or the service user does not want to disclose. |

## Special GP Practice ODS Codes

| Code | Meaning |
| --- | --- |
| `V81997` | No Registered GP practice |
| `V81998` | GP Practice Code not applicable |
| `V81999` | GP Practice Code not known |

## Validation Notes

- `providerUbrn` uniquely identifies the referral submission sent by DWMP.
- Date values are date-only strings in `yyyy-MM-dd` format and do not include time or timezone offset values.
- `mobile` and `telephone` are conditionally required. At least one will be supplied.
- Strings will be sent as `null` when no value is available.
- Boolean values may be `true`, `false`, or `null`. A `null` value means unknown or not disclosed.
- Enum-like fields will use the exact token values shown in this document.
- Numeric values will be submitted as JSON numbers, not strings.

## Example DWMP Request

This is an example request body that DWMP may send to a provider API.

```json
{
  "idempotencyKey": "41b93a15-4e47-4513-99bf-d3a332548dd1",
  "providerUbrn": "123456789012",
  "dateOfReferral": "2026-05-27",
  "providerSelectedDate": "2026-05-27",
  "referralSource": "GP",
  "address1": "1 High Street",
  "address2": null,
  "address3": null,
  "age": 45,
  "bmi": 31.25,
  "bmiDate": "2026-05-20",
  "email": "service.user@example.com",
  "ethnicity": "White",
  "familyName": "User",
  "givenName": "Service",
  "gpPracticeOdsCode": "A1B234",
  "height": 172.50,
  "mobile": "+447123456789",
  "postcode": "AB1 2CD",
  "sexAtBirth": "Female",
  "telephone": null,
  "triagedLevel": 2,
  "hasArthritisOfHip": null,
  "hasArthritisOfKnee": false,
  "hasDiabetesType1": false,
  "hasDiabetesType2": true,
  "hasHypertension": null,
  "hasLearningDisability": false,
  "hasPhysicalDisability": false,
  "hasRegisteredSeriousMentalIllness": null,
  "isVulnerable": false
}
```

## Provider Success Response

Once the provider's API has successfully validated and stored the referral resource, it should return `201 Created`.

### Payload Fields

The following fields in the provider success response will be processed.

| Field | Data Type | Required | Max Length | Format / Allowed Values | Description |
| --- | --- | --- | --- | --- | --- |
| `providerUbrn` | `string` | Yes | 12 | `^\d{12}$` | Unique booking reference number. |
| `receivedAtUtc` | `string` | Yes | 20 | `yyyy-MM-ddTHH:mm:ssZ` | The UTC date and time the referral was received by the provider API. |
| `providerReferralId` | `string` | No | 200 |  | Optional field for the provider to supply their referral identity to aid correlation. |


### Response Example

This is an example success response body that the provider API should return to DWMP.

```http
HTTP/1.1 201 Created
Location: /api/referral/123456789012
Content-Type: application/json
```

```json
{
  "providerUbrn": "123456789012",
  "providerReferralId": "PR-000123",
  "receivedAtUtc": "2026-05-27T09:36:42Z"
}
```

## Provider Validation Error Response

If any of the request payload fields fail validation the provider's API should return `400 Bad Request` with a content type of `application/problem+json`.

### Payload Fields

| Field | Data Type | Required | Max Length | Format / Allowed Values | Description |
| --- | --- | --- | --- | --- | --- |
| `type` | `string` | No | 500 | URL | Link to RFC9110's section describing problem type. |
| `title` | `string` | No | 500 | | Problem title. |
| `status` | `integer` | No |  | 400 | 400 Bad Request status code. |
| `errors` | `object` | Yes |  |  | Fields returned in the errors object should match the payload field names, and each field should have an array of strings describing the validation error for that field.  |


### Response Example 

This is an example validation failure response body that the provider API should return to DWMP.

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
```

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "providerUbrn": [
      "Value must match ^\\d{12}$."
    ],
    "mobile": [
      "Either mobile or telephone must be supplied."
    ],
    "telephone": [
      "Either mobile or telephone must be supplied."
    ]
  }
}
```

## Provider Conflict Error Response

 If a referral with the same `providerUbrn` has already been created, either on a previous occasion or following a timeout and a re-request, the provider's API should return `409 Conflict` with a content type of `application/problem+json`.

### Payload Fields

| Field | Data Type | Required | Max Length | Format / Allowed Values | Description |
| --- | --- | --- | --- | --- | --- |
| `type` | `string` | No | 500 | URL | Link to RFC9110's section describing problem type. |
| `title` | `string` | No | 500 | | Problem title. |
| `status` | `integer` | No |  | 409 | 409 Conflict status code. |
| `error` | `string` | Yes | 500  |  | Description of why the 409 occurred. |
| `providerUbrn` | `string` | Yes | 12 | `^\d{12}$` | Unique booking reference number. |
| `receivedAtUtc` | `string` | Yes | 20 | `yyyy-MM-ddTHH:mm:ssZ` | The UTC date and time the referral was received by the provider API. |
| `providerReferralId` | `string` | No | 200 |  | Optional field for the provider to supply their referral identity to aid correlation. |

### Example Provider Conflict Error

```http
HTTP/1.1 409 Conflict
Location: /api/referral/123456789012
Content-Type: application/problem+json
```

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.10",
  "title": "Conflict.",
  "status": 409,
  "error": "Referral already exists.",
  "providerUbrn": "123456789012",
  "providerReferralId": "PR-000123",
  "receivedAtUtc": "2026-05-27T09:36:42Z"  
}
```
