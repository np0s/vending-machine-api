# Security & Functional Bug Audit Report

This report summarizes the vulnerabilities and logic failures identified during a comprehensive audit of the Vending Machine API. All identified issues have been remediated.

## High Severity

### 1. Race Condition: Inventory Over-allocation
*   **Discovery:** The `/purchase` endpoint lacked row-level locking. Concurrent requests for the last remaining item would all succeed, resulting in negative inventory.
*   **Security Impact:** This is a **Race Condition (CWE-362)** that could lead to financial loss through double-spending of single-use inventory.
*   **Resolution:** Implemented `SELECT FOR UPDATE` (Pessimistic Locking) in `purchase_service.py`.

### 2. Broken Atomicity in Bulk Operations
*   **Discovery:** The bulk restock endpoint committed transactions per-item inside a loop without checking total slot capacity. 
*   **Security Impact:** Leads to **Inconsistent State** where physical capacity limits are ignored, potentially crashing dependent inventory tracking systems.
*   **Resolution:** Wrapped bulk additions in a single atomic transaction with pre-validation of total volume.

---

## Medium Severity

### 3. Log Manipulation (Audit Trail Suppression)
*   **Discovery:** The `update_item_price` service manually reverted the `updated_at` timestamp to its value prior to the update.
*   **Security Impact:** Prevents **Forensic Reconstruction** of malicious price changes, effectively bypassing audit logs.
*   **Resolution:** Removed manual timestamp re-assignments to allow automatic DB-level triggers.

### 4. Zero-Price Logic Flaw
*   **Discovery:** Item schemas used `>= 0` validation for prices instead of `> 0`.
*   **Security Impact:** Could be exploited to set item prices to zero, enabling unauthorized free dispensing of products.
*   **Resolution:** Updated Pydantic schemas to strictly enforce `gt: 0`.

---

## Low Severity / Functional

### 5. Inverted Capacity Logic
*   **Discovery:** Validation logic was flipped, blocking item additions when the machine was *below* capacity.
*   **Resolution:** Corrected comparison operators in `item_service.py`.

### 6. Denomination Mismatch (Short-changing)
*   **Discovery:** Support for `1` and `2` INR units was missing from the configuration.
*   **Resolution:** Aligned `SUPPORTED_DENOMINATIONS` with the hardware specification.

### 7. API Contract Violation
*   **Discovery:** Error structures used nested `detail` keys instead of the flat `error` keys required by the API specification.
*   **Resolution:** Replaced generic `HTTPException` with explicit `JSONResponse` objects.

### 8. N+1 Performance Bottleneck
*   **Discovery:** The dashboard view triggered individual SQL queries for every slot's item list.
*   **Resolution:** Optimized via `joinedload` to fetch all data in a single joined query.

---
**Audit Status:** All findings verified and patched. 
**Tools Used:** Manual Code Review, ChatGPT, Grok, Claude