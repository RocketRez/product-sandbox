# Photo review instrumentation

Logging for membership photo validation, so RocketRez can tune default check thresholds and suggested denial reasons, and adjust individual tenants based on how their staff review photos.

The prototype (`membership-photo-review.html`) emits these events and shows the derived metrics on its Insights tab.

## Goals

1. **Check accuracy.** For each automatic check, measure how often staff agree when it sends a photo to review, and how often it lets a bad photo through. Use this to set thresholds and the "when it fails / when unsure" actions, by default and per tenant.
2. **Suggestion accuracy.** For each suggested denial reason, measure how often staff accept it, change it, approve the photo instead, or edit the message sent to the guest. Use this to fix how evidence maps to reasons, and to improve default guest messages.

## Common fields

Every event carries:

| Field | Type | Notes |
|---|---|---|
| `event` | string | Event name |
| `event_id` | string | Unique per event |
| `ts` | ISO 8601 | Server time |
| `tenant_id` | string | Operator, e.g. `lagoon` |

Use pseudonymous IDs only (`photo_id`, `member_id`, `reviewer_id`). Never log names, email addresses or photo pixels in analytics events.

## Events

### `photo_checked`

Sent once per photo after the automatic checks run.

| Field | Type | Notes |
|---|---|---|
| `photo_id` | string | Stable for the life of the photo |
| `member_id` | string | |
| `product` | string | Membership product |
| `source` | enum | `member_portal`, `gate`, `pos` |
| `approval_mode` | enum | `assisted`, `manual`, `gate` |
| `model_version` | string | Face model build |
| `rules_version` | string | Threshold and action config in effect |
| `checks[]` | array | `{check, score 0–100, status ok/warn/bad, threshold}` for all 8 checks |
| `outcome` | enum | `auto_approved`, `returned_to_guest`, `sent_to_review`, `verify_at_gate` |
| `suggested_reason` | string \| null | Reason the system would suggest to staff |

### `review_decision`

Sent when staff approve or deny a photo in the review queue or from a spot check.

| Field | Type | Notes |
|---|---|---|
| `photo_id` | string | Joins to `photo_checked` |
| `reviewer_id` | string | |
| `source` | enum | `review_queue`, `spot_check` |
| `decision` | enum | `approve`, `deny` |
| `suggested_reason` | string \| null | What the panel suggested |
| `final_reason` | string \| null | Reason the photo was denied with |
| `suggestion` | enum | `accepted` (denied with the suggestion), `overridden` (denied with another reason), `rejected` (approved despite a suggestion), `none` |
| `message_edited` | bool | Staff changed the message sent to the guest |
| `message_template` | string \| null | Reason whose message was sent |
| `flagged_checks[]` | array | `{check, score}` for checks that were not `ok` |
| `time_to_decision_ms` | int | From the photo opening to the decision |
| `rules_version` | string | |

Store the edited message text in the guest communication record, not in this event.

### `review_reopened`

Sent when staff undo or reopen a decision. Analytics drops the earlier `review_decision` for that `photo_id` and keeps the next one.

| Field | Type |
|---|---|
| `photo_id`, `reviewer_id` | string |
| `previous_decision`, `previous_reason` | string |

### `spot_check_completed`

Sent when staff finish the daily sample of auto-approved photos (at most 5 a day).

| Field | Type | Notes |
|---|---|---|
| `reviewer_id` | string | |
| `sampled` | int | Photos in the sample |
| `flagged_photo_ids[]` | array | Photos sent back to review. Their `review_decision` (source `spot_check`) gives the reason that was missed |

### `gate_photo_action`

| Field | Type | Notes |
|---|---|---|
| `member_id`, `product` | string | |
| `action` | enum | `photo_captured`, `photo_confirmed`, `photo_mismatch` |

`photo_mismatch` in gate-check mode is a miss for the automatic checks, just as a flagged spot check is.

### `setting_changed`

| Field | Type | Notes |
|---|---|---|
| `setting` | string | e.g. `threshold.hat`, `checks.light.unsure`, `spot_check.daily_sample`, `reasons.glasses.message` |
| `from`, `to` / `value` | any | |
| `source` | enum | `settings`, `insights_recommendation`, `insights_reset` |

## Metrics (30-day window, per tenant and across all tenants)

**Check accuracy**, per check:

- **Sent to review:** `review_decision` rows where the check is in `flagged_checks`.
- **Staff agreed:** of those, the share denied with the reason that check maps to (eyes → Sunglasses, light → Too dark, sharp → Blurry, hat → Hat or face covering, size/pose/face → Face too small or turned, one → More than one person).
- **Missed:** spot-check photos denied with the check's reason, plus gate mismatches, divided by photos sampled.

**Suggestion accuracy**, per suggested reason:

- Accepted, overridden (and to which reason), approved instead, and message edited, each as a share of times suggested.

## Recommendation rules (prototype starting point)

Recommendations need at least 50 flagged photos for a check, or at least one spot-check miss.

| Signal | Recommendation |
|---|---|
| Missed ≥ 1.5% of spot-checked photos | Raise the flag threshold by 5 |
| Staff agreed < 45% | Lower the flag threshold by 8 |
| Staff agreed ≥ 80% and the unsure action is not already "retake" | Ask the guest to retake when unsure, so staff no longer see these |
| A suggestion is changed to the same other reason ≥ 15% of the time | Review the evidence-to-reason mapping |
| A suggestion's photo is approved instead ≥ 50% of the time | Review the check's threshold |
| A message is edited ≥ 25% of the time | Update the default message |

Applying a recommendation to a tenant writes a tenant override, logs `setting_changed` with `source: insights_recommendation`, and shows as "Applied · collecting data" until a new 30-day window has built up. Reset returns the tenant to the default.

## Changing defaults vs. tenant overrides

- **Defaults** change when the same recommendation shows up across most tenants with enough volume. Bump `rules_version`.
- **Tenant overrides** cover operator-specific standards (for example a tenant that allows caps). They are visible on the Insights tab and reversible with Reset.
- Compare metrics before and after each change by `rules_version`. Roll back if staff agreement or the miss rate gets worse.

## Open questions

- Who can apply recommendations: RocketRez only, or operator admins too?
- How long to keep `photo_checked` scores, and whether they may be used later to train a model (needs a policy with operators, since many photos are of children).
- Whether to add an explicit "suggestion was wrong" flag to approvals, in addition to the inferred `rejected`.
