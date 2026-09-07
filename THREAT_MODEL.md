# django-enumfields threat model

## Overview

A Django field library converts application enum members to database scalar values and back, with form/admin integration and optional Django REST Framework fields. Assignment conversion, serialization and DRF parsing are library-owned; routing, endpoint authentication and permission to change an enum-valued business state belong to the caller (enumfields/fields.py:14; enumfields/drf/fields.py:6).

The protected boundary is conversion between host-defined enum types and values supplied by forms, APIs or the database. The library cannot infer which user may set a particular member, whether a transition is legal, or which objects belong to a user. Those are domain authorization rules. Its representation choices also form a compatibility contract: changing enum values or enabling alternate name output can change what existing clients and stored rows mean, even when every individual value is syntactically valid.

| Component | Source |
| --- | --- |
| Model assignment, conversion and storage | enumfields/fields.py:14; enumfields/fields.py:53; enumfields/fields.py:147 |
| DRF input/output adapter | enumfields/drf/fields.py:6; enumfields/drf/fields.py:34 |
| Form integration and package | enumfields/fields.py:119; setup.py:30 |

| Deployment or workflow | Resource or capability | Configuration and precedence | Safe effective value or location | Readers, writers, or recipients | Enforcing control | Evidence or unknowns |
| --- | --- | --- | --- | --- | --- | --- |
| Library embedded in Django | Database field preparation | Caller enum/model field → assignment/conversion → variant get_prep_value → host ORM | Host model CharField or IntegerField column; scalar enum representation | Host DB and later ORM readers | Library conversion plus caller schema/authorization | enumfields/fields.py:130; enumfields/fields.py:147 |
| Optional DRF integration | DRF serialization | Host DRF serializer → enum + ints_as_names/lenient options → representation | Enum value, or lowercase name for integer enum with ints_as_names enabled | Host API clients | DRF choice validation and host view permissions | enumfields/drf/fields.py:21 |

## Threat Model, Trust Boundaries, and Assumptions

**Protected assets.** Enum/scalar consistency in model storage and API representation (enumfields/fields.py:53; enumfields/fields.py:75). Business state represented by enum values, whose permitted transitions are application-specific.

**Actors and starting authority.** A user of a host API may submit arbitrary scalar input, including a valid but unauthorized state; enum membership alone does not authorize it. An application developer choosing the enum/import path already holds code configuration authority.

**Trust boundaries and owned controls.**

- Trusted enum classes or configured dotted import paths define allowed values; model values pass through the assignment descriptor and to_python, which raises ValidationError for unmatched nonempty input. Enum import configuration is code authority, not a user-selectable data format (enumfields/fields.py:30; enumfields/fields.py:35).
- DB reads invoke to_python; writes use field preparation. Char and integer variants differ, with EnumIntegerField accepting int conversion in preparation. The library supplies conversion, not a database state-transition authorization constraint (enumfields/fields.py:65; enumfields/fields.py:147).
- DRF input uses choice-string mapping with optional lenient case/name/value matching; output can map integer enums to lowercase names. Host serializer endpoints must still limit object access and allowed transitions (enumfields/drf/fields.py:21; enumfields/drf/fields.py:34).

**Security objectives.** Reject invalid representations at the intended input boundary and retain stable serialized meanings across migrations. Apply application authorization and transition rules independently of enum conversion. Keep trusted enum import paths under code/configuration control.

**Assumptions and unresolved controls.**

- Database engine/schema constraints and actual DRF endpoints are absent, so no host-specific request exposure or DB enforcement is asserted.
- Setuptools packaging identifies django-enumfields 1.0.0; no publisher workflow appears in inventory (setup.py:30).
- No host schema migration history, DB constraints, object ownership, endpoint definition or network/log sink is supplied. Library defaults cannot establish those controls. Enum import strings are trusted model configuration; scalar caller input does not inherently reach import_string.

## Attack Surface, Mitigations, and Attacker Stories

These are prioritized hypotheses, not validated vulnerabilities. Each requires its stated caller, data and exposure prerequisites; ordinary use of authority already granted is not a new capability.

| Priority | Scenario and capability gain | Prerequisites | Impact | Existing controls | Mitigation | Evidence |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | A user submits a valid enum member representing a business state they are not allowed to enter. | Host endpoint equates valid choice parsing with transition permission. | Unauthorized state or privilege change, limited by the host domain. | Enum conversion/DRF choices reject some invalid representations; they do not authorize transitions. | Apply object, field and transition authorization after parsing and before persistence. | enumfields/fields.py:53; enumfields/drf/fields.py:34 |
| 2 | Different input/storage paths give inconsistent treatment to a scalar and a host assumes all writes are membership constrained. | A host uses integer preparation or another ORM path under a security-relevant membership invariant. | Invalid or misinterpreted state reaching a later consumer. | Assignment conversion and DRF mapping; integer preparation separately accepts int conversion. | Define and verify the actual DB/input contract and add database constraints when the domain requires them. | enumfields/fields.py:30; enumfields/fields.py:147 |
| 2 | An API or migration changes representation so an old client interprets a valid value as a different state. | Enum definitions evolve or ints_as_names/lenient mode changes across a compatibility boundary. | Incorrect business action or unintended acceptance of aliases. | Explicit enum class, serializer options and stable scalar conversion paths. | Version meaningful enum changes and preserve unambiguous transition semantics. | enumfields/fields.py:75; enumfields/drf/fields.py:21; enumfields/drf/fields.py:44 |
| 3 | An integration accepts an untrusted dotted enum-class path as data and imports it. | A caller exposes model-field configuration across a genuine lower-trust boundary; not supplied by this package. | Execution of selected import code with host authority. | Import path is a constructor configuration option, not a DRF data field. | Keep enum class/import selection in trusted source and deployment configuration. | enumfields/fields.py:35 |

## Severity Calibration (Critical, High, Medium, Low)

| Level | Repository-specific example | Counterexample or limiting prerequisite |
| --- | --- | --- |
| Critical | A separately demonstrated host configuration boundary allows broad privileged code execution through import selection. | No remote class-path selection is established; developers specifying enum classes already write executable Python. |
| High | A host accepts a valid enum value that grants an unauthorized privileged business state. | Needs actual business authority and missing host transition enforcement, not merely successful enum parsing. |
| Medium | Representation inconsistency causes a meaningful but bounded integrity error in a real persisted workflow. | Prove the input path and downstream interpretation; invalid values rejected early are counterevidence. |
| Low | Malformed input produces a validation error or localized display incompatibility without protected-state impact. | Choosing lenient parsing intentionally is not itself a security defect. |

This model uses an independent source-backed architecture pass. Repository citations were checked against the supplied inventory and source lines; application code and external services were not executed. Source-established behavior is distinct from unverified deployment exposure. Revisit the model when the described input, storage, authorization or publication boundaries change.

Repository: github.com/mathspace/django-enumfields
Version: 169609c9f37914430c8481df85a66660af6e8750
