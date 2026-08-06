# Task1 POC test cases

## Transfer

```json
{"request_id":"REQ-TRANSFER-001","employee_id":"E1001","change_type":"transfer","after_department_code":"FIN","after_position_code":"staff","mode":"dry_run"}
```

Expected add: `FIN_VIEW`, `FIN_APPROVE` / remove: `HR_VIEW`, `HR_EDIT`.

## Promotion

```json
{"request_id":"REQ-PROMOTION-001","employee_id":"E1001","change_type":"promotion","after_department_code":"HR","after_position_code":"manager","mode":"dry_run"}
```

Expected add: `APPROVAL_MANAGER`.

## Termination

```json
{"request_id":"REQ-TERM-001","employee_id":"E9001","change_type":"termination","after_employment_status":"terminated","mode":"dry_run"}
```

Expected `target_permissions: []`.

## Unknown department

```json
{"request_id":"REQ-UNKNOWN-DEPT-001","employee_id":"E1001","change_type":"transfer","after_department_code":"LEGAL","after_position_code":"staff","mode":"dry_run"}
```

Expected `status: review_required`.

## Apply test

```json
{"request_id":"REQ-APPLY-TEST-001","employee_id":"E1001","change_type":"transfer","after_department_code":"FIN","after_position_code":"staff","mode":"apply_test"}
```

Expected: only `user_permissions_test` changes.
