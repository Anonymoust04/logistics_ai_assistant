# Logistics AI Assistant: Final Database Entities

This is the finalised data model for the project. It starts from the normalised core (Parcel, Client, Hub, Staff, Transport, Delivery and the tracking tables) and applies the improvements raised during review: explicit keys and nullability, a clear grain for Delivery, an event history that can say what happened and where, one source of truth for each fact, and documented business rules.

**Conventions**

- Names use `snake_case`, and every identifier ends in `_id`, so foreign keys read the same everywhere.
- PK = primary key, FK = foreign key, "Null" = whether the column may be empty.
- All data is synthetic. No real customer, staff or shipment data is used.


## 1. Custom data types

| Type | Values |
|---|---|
| `hub_type` | AIRPORT, PORT, OFFICE, SORTING_CENTRE |
| `address_type` | HOME, BUSINESS, THIRD_PARTY_OFFICE, HUB |
| `client_type` | PERSON, BUSINESS |
| `service_type` | EXPRESS, STANDARD |
| `delivery_status` | WAITING, TRANSPORTING, ARRIVED, FAILED, POSTPONED |
| `event_type` | PICKED_UP, ARRIVED_AT_HUB, DEPARTED_HUB, CUSTOMS_HOLD, DELAYED, OUT_FOR_DELIVERY, DELIVERED, DELIVERY_FAILED |
| `delay_reason` | WEATHER, CUSTOMS, MECHANICAL, ADDRESS_ISSUE, OTHER |

`address_type` describes the **place itself**, so it is stored on Address and is not derived from whether a hub points to it.


## 2. Entities

### country
| Column | Key | Null | Notes |
|---|---|---|---|
| country_code | PK | No | Two-letter code |
| country_name | | No | |
| region | | No | |

### postcode
| Column | Key | Null | Notes |
|---|---|---|---|
| postcode | PK | No | Part of composite key |
| country_code | PK, FK (country) | No | Part of composite key |
| area | | No | |
| city | | No | |
| district | | No | |
| state | | No | |

### address
| Column | Key | Null | Notes |
|---|---|---|---|
| address_id | PK | No | |
| address_type | | No | |
| unit | | Yes | Floor or unit, e.g. "Level 27" |
| building | | Yes | |
| street_number | | Yes | |
| street | | No | |
| postcode | FK (postcode) | Yes | Not every place has one |
| country_code | FK (country, and postcode with postcode) | No | Always present |
| latitude | | Yes | Optional, for believable vehicle movement |
| longitude | | Yes | Optional |

The pair (`postcode`, `country_code`) references `postcode`. Because `country_code` is also stored in the same row, one column carries both the country link and the postcode link, so they cannot disagree.

### client
| Column | Key | Null | Notes |
|---|---|---|---|
| client_id | PK | No | |
| client_type | | No | Person or business |
| name | | No | Person or company name |
| contact_name | | Yes | Contact person for a business |
| email | | No | |
| phone | | No | |
| address_id | FK (address) | No | |

### parcel
| Column | Key | Null | Notes |
|---|---|---|---|
| parcel_serial_id | PK | No | |
| sender_client_id | FK (client) | No | |
| receiver_client_id | FK (client) | No | |
| weight_kg | | No | |
| fragile | | No | |
| contents_description | | No | Supports customs and policy questions |
| declared_value | | Yes | |
| service_type | | No | Affects delivery expectations |

### hub
| Column | Key | Null | Notes |
|---|---|---|---|
| hub_id | PK | No | |
| name | | No | |
| email | | No | |
| phone | | No | |
| hub_type | | No | |
| address_id | FK (address) | No | Address type must be HUB |
| unlocode | | Yes | UN/LOCODE, e.g. MYKUL |

### staff
| Column | Key | Null | Notes |
|---|---|---|---|
| staff_id | PK | No | |
| name | | No | |
| email | | No | |
| phone | | No | |
| hub_id | FK (hub) | No | |
| role | | No | |

### transport
| Column | Key | Null | Notes |
|---|---|---|---|
| transport_id | PK | No | |
| transport_type | | No | |
| registration_number | Alternate key (unique) | No | Natural second key |
| registration_date | | No | |
| last_maintenance_date | | Yes | Replaces the ambiguous "year of maintenance" |

### delivery
One row is **one parcel's leg on one vehicle**.

| Column | Key | Null | Notes |
|---|---|---|---|
| delivery_id | PK | No | |
| parcel_serial_id | FK (parcel) | No | |
| leg_number | | No | Unique together with `parcel_serial_id` |
| transport_id | FK (transport) | No | |
| driver_staff_id | FK (staff) | Yes | Empty for legs without a single driver |
| status | | No | Owns whether the leg is finished |
| origin_address_id | FK (address) | No | |
| destination_address_id | FK (address) | No | |
| departure_time | | No | Planned or actual |
| expected_arrival_time | | No | |
| actual_arrival_time | | Yes | Required only when status is ARRIVED |

### tracking_event
A history table: one row per parcel event.

| Column | Key | Null | Notes |
|---|---|---|---|
| event_id | PK | No | |
| parcel_serial_id | FK (parcel) | No | |
| delivery_id | FK (delivery) | Yes | Empty for events outside a leg, e.g. a hub scan |
| hub_id | FK (hub) | Yes | Where the event happened, when at a hub |
| event_type | | No | |
| event_timestamp | | No | Orders events in time |
| delay_reason | | Yes | Only for DELAYED events |

### coordinates
| Column | Key | Null | Notes |
|---|---|---|---|
| transport_id | PK, FK (transport) | No | |
| recorded_at | PK | No | A value, not a foreign key |
| latitude | | No | |
| longitude | | No | |

### speed
| Column | Key | Null | Notes |
|---|---|---|---|
| transport_id | PK, FK (transport) | No | |
| recorded_at | PK | No | |
| speed_kmh | | No | |

Coordinates and speed stay in separate tables because they are assumed to arrive from the vehicle at different rates. If one device always reports both together, they can be merged into one table with the same composite key.


## 3. Business rules and constraints

1. **Delivery grain.** One delivery row is one parcel on one vehicle for one leg. Vehicle details repeat when a vehicle carries several parcels, which is accepted for this project.
2. **Leg ordering.** `(parcel_serial_id, leg_number)` is unique. A failed leg is replaced by a new row with a higher leg number.
3. **Status owns completion.** `actual_arrival_time` must be filled when status is ARRIVED and empty otherwise.
4. **Current delivery is derived, never stored.** A parcel's current delivery is the leg with status WAITING, TRANSPORTING or POSTPONED that has the earliest `departure_time`, with ties broken by `leg_number`. If no leg matches, the parcel is either delivered or not yet scheduled, and the latest tracking event tells which.
5. **Address type and hubs.** A hub's address must have type HUB.
6. **Hub country.** The first two characters of `unlocode` must equal the country of the hub's address. The hub stores no separate country column.
7. **Location can host several hubs.** `unlocode` is not unique and is not a key.
8. **Postcode simplification.** Within a country, a postcode determines its area, city, district and state. Real postcodes can cross boundaries, so this is a simplification for synthetic data.
9. **Tracking event consistency.** When `delivery_id` is filled, that delivery's parcel must equal `parcel_serial_id`. Both columns exist because some events have no delivery.
10. **Event details.** `delay_reason` may be filled only when `event_type` is DELAYED. No two events for the same parcel may share the same `event_type` and `event_timestamp`.
11. **Positions belong to vehicles**, not parcels. A parcel's position in transit is found through its current delivery.
12. **Privacy.** Client, staff and address details are excluded from the allow-list of the natural-language-to-SQL tool.

## 4. Relationships

Each link was tested with two questions: can one A relate to many B, and can one B relate to many A?

| Link | Result | Notes |
|---|---|---|
| Client to parcel (sender) | 1:N | |
| Client to parcel (receiver) | 1:N | |
| Parcel to transport | **M:N** | Resolved by delivery |
| Staff to transport | **M:N** | Resolved by delivery |
| Parcel to delivery | 1:N | A parcel can have several legs |
| Transport to delivery | 1:N | |
| Staff to delivery | 1:N | Optional driver |
| Hub to staff | 1:N | |
| Parcel to tracking event | 1:N | |
| Delivery to tracking event | 1:N | Optional |
| Hub to tracking event | 1:N | Optional |
| Transport to coordinates / speed | 1:N | Streams |
| Address to client / hub | 1:N | |
| Address to delivery (origin, destination) | 1:N | Two separate references |
| Postcode to address | 1:N | Reference is the pair (postcode, country code) |
| Country to postcode | 1:N | |



## 5. Review points addressed

| Review point | Resolution |
|---|---|
| Tracking event could not say what or where | Added `event_type`, `hub_id`, `delay_reason`, and a consistent timestamp |
| "Current" in column names of a history table | Removed. Current state is derived |
| Two sources of truth for completion | Status owns it (rule 3) |
| Delivery grain unclear | One parcel leg per row, with `leg_number` |
| Origin and destination unexplained | Both reference address |
| Hub country duplicated in `unlocode` | Country not stored on hub (rule 6) |
| Postcode and country link | Composite foreign key from address to postcode |
| Missing postcode case | `postcode` is nullable and `country_code` is always present |
| Optional driver | `driver_staff_id` is nullable |
| Timestamp labelled as a foreign key | `recorded_at` is part of the primary key only |
| Ambiguous "year of maintenance" | `last_maintenance_date`, plus a unique `registration_number` |
| Business premises and companies | `BUSINESS` address and client types, optional `contact_name` |
| Dropped columns without reasons | Section 7 |
| Postcode hierarchy | Documented as an accepted deviation |
| Inconsistent names and keys | `snake_case`, `_id` suffix and explicit key and null labels |