# Performance Analysis Report - Ghanem Production (Odoo 16)
# Phase 1 :

**Date:** 2026-02-16
**Environment:** Odoo.sh Production (bwtco-ghanem-production-8981920)
**Profile Source:** (Flame Graph - 12,577 total samples)
**Focus Area:** Custom Accounting Modules

---

## Executive Summary

A system-wide CPU profiling session captured 12,577 samples across all Odoo worker processes. Analysis of the flame graph reveals that custom module overrides on `account.move` and `account.move.line` are responsible for approximately **18-20% of total CPU time**. The three major bottleneck areas are:

1. **action_post override chain** - 6 custom modules forming a deep call chain on every invoice/bill/entry post (9.44%)
2. **account.move.line create/write cascade** - Record-by-record field assignments triggering cascading write calls (8.5%)
3. **Payslip analytic distribution lookup** - Per-key SQL queries during move line creation (0.6%)

---

## Bottleneck #1: action_post Override Chain

**Impact:** 9.44% of total CPU (1,187 samples)
**Severity:** Critical
**Affected Operation:** Posting any invoice, bill, credit note, or journal entry

### Description

Six custom modules override `account.move.action_post()`, creating a deep super() call chain. Every single document post runs through ALL six overrides regardless of document type:

| Order | Module | File | Line | Purpose |
|-------|--------|------|------|---------|
| 1 | restrict_invoicing | models/account_move.py | 185 | Validates refund quantities against invoiced/delivered quantities |
| 2 | government_incentive | models/account_move.py | 33 | Creates incentive journal entry, posts it, and reconciles |
| 3 | custom_discount_module | models/models.py | 788 | Shows warning popup for vendor bills |
| 4 | sale_returns_management | models/account_move.py | 44 | Validates refund quantities against invoiced quantities |
| 5 | journal_audit | models/account_move.py | 33 | Sets review_state for journal entries with expense accounts |
| 6 | ghanim_custome_accounting | models/account_move.py | 14 | Validates analytic distribution on expense journal entries |

After these six, the call continues into Odoo core and enterprise modules:
- sale/models/account_move.py
- account_accountant/models/account_move.py
- account/models/account_move.py (_post)
- stock_account, purchase_stock, l10n_sa, account_reports, stock_landed_costs, account_asset

### Issue 1A: Recursive action_post in government_incentive

**File:** `government_incentive/models/account_move.py:33-86`
**Severity:** Critical

When an outgoing invoice has a government incentive amount, the module creates a new journal entry and calls `action_post()` on it:

```python
def action_post(self):
    res = super(AccountMove, self).action_post()
    if self.move_type == 'out_invoice' and self.incentive_amount > 0 ...:
        # ... creates journal entry ...
        journal_entry = self.env['account.move'].create({...})
        journal_entry.action_post()  # RECURSIVE - runs entire chain again
        # ... reconcile ...
    return res
```

This means the entire 6-module override chain runs **twice** for every invoice that has an incentive. The inner `action_post()` call goes through restrict_invoicing, custom_discount_module, sale_returns_management, journal_audit, and ghanim_custome_accounting again - all unnecessarily since it is a simple journal entry, not a refund or vendor bill.

**Recommendation:** Post the incentive journal entry with context flags to skip unnecessary validations, or call `_post()` directly with appropriate safeguards.

### Issue 1B: Duplicate Refund Validation Logic

**Files:**
- `restrict_invoicing/models/account_move.py:175-185`
- `sale_returns_management/models/account_move.py:18-44`

**Severity:** High

Both modules validate that refund quantities do not exceed invoiced quantities for `out_refund` moves. They use different but overlapping approaches:

**restrict_invoicing** (`is_valid_credit_note()`):
- Gets sale orders from invoice lines
- Gets all invoice_ids from those sale orders
- Iterates through all invoices counting invoiced/credited quantities
- Also checks delivery picking returns
- Also checks credit note date limits

**sale_returns_management** (`action_post()`):
- For each refund line, gets sale_line_ids
- For each sale_line, calls `_get_invoice_lines()` and filters posted ones
- Counts invoiced vs refunded quantities per line

Both run for every `out_refund` post, performing redundant database queries on the same sale orders, invoices, and picking records.

**Recommendation:** Consolidate refund validation into a single module. Remove the duplicate logic from one of the two modules.

### Issue 1C: Warning Popup Blocking Every Vendor Bill

**File:** `custom_discount_module/models/models.py:770-788`
**Severity:** Medium

For every `in_invoice` and `in_refund`, the module checks if `exclude_from_contract` is False and if `pass_warning` context is not set. If so, it returns a warning popup action instead of posting:

```python
def action_post(self):
    for rec in self:
        if not rec.exclude_from_contract and not self.env.context.get('pass_warning') and rec.move_type in ('in_invoice', 'in_refund'):
            # Returns warning popup instead of posting
            return action
    super(AccountMove, self).action_post()
```

This forces every vendor bill post to require **two round-trips**: first to show the warning, then to actually post (with `pass_warning` context). This doubles the overhead for vendor bill posting.

**Recommendation:** Consider making this warning optional or only showing it when the bill is actually linked to a discount contract.

### Issue 1D: Unnecessary Iteration for Non-Matching Move Types

**Files:**
- `journal_audit/models/account_move.py:21-33`
- `ghanim_custome_accounting/models/account_move.py:8-14`

**Severity:** Low-Medium

Both modules iterate through `self.line_ids` but only when `move_type == 'entry'`. However, the override still runs (enters the method, checks conditions) for all move types including invoices, bills, and refunds.

```python
# journal_audit
def action_post(self):
    if self.move_type == 'entry' and ...:
        for line in self.line_ids:  # Only runs for entries
            ...
    return super().action_post()

# ghanim_custome_accounting
def action_post(self):
    if self.move_type == 'entry':
        for line in self.line_ids:  # Only runs for entries
            ...
    return super().action_post()
```

**Recommendation:** While each individual check is lightweight, having 6 overrides adds method call overhead. Consider consolidating these entry-type validations into a single module.

---

## Bottleneck #2: account.move.line create/write Cascade

**Impact:** ~8.5% of total CPU (360+ samples per layer)
**Severity:** Critical
**Affected Operation:** Creating or modifying invoice lines (happens during posting, invoice creation, and line edits)

### Issue 2A: invoice_sellout - Per-Record Field Assignments in create()

**File:** `invoice_sellout/models/account_move_line.py:11-19`
**Severity:** Critical

```python
@api.model_create_multi
def create(self, vals_list):
    res = super(AccountMoveLine, self).create(vals_list if vals_list else [])
    for rec in res:
        if rec.product_id:
            rec.basic_price = rec.product_id.basic_price              # Line 16 - triggers write()
            rec.last_basic_price_update = rec.product_id.last_basic_price_update  # Line 17 - triggers write()
            rec.sellout_price = rec.product_id.sellout_price          # Line 18 - triggers write()
    return res
```

Each `rec.field = value` assignment triggers a separate `write()` call to the database. For an invoice with N lines, this generates **3 x N individual write() calls** after the initial batch create. Each write() call then cascades through the multi_branches write() override (Issue 2B).

The flame graph confirms this: lines 16, 17, and 18 each appear as separate hot spots with 360, 353, and 341 samples respectively.

**Recommendation:** Set these values in `vals_list` BEFORE calling `super().create()`:

```python
@api.model_create_multi
def create(self, vals_list):
    Product = self.env['product.product']
    product_ids = set()
    for vals in vals_list:
        if vals.get('product_id'):
            product_ids.add(vals['product_id'])
    products = {p.id: p for p in Product.browse(list(product_ids))}
    for vals in vals_list:
        pid = vals.get('product_id')
        if pid and pid in products:
            product = products[pid]
            vals['basic_price'] = product.basic_price
            vals['last_basic_price_update'] = product.last_basic_price_update
            vals['sellout_price'] = product.sellout_price
    return super().create(vals_list if vals_list else [])
```

### Issue 2B: multi_branches - Per-Record branch_id Assignment in create() and write()

**File:** `multi_branches/models/account_invoice.py:87-100`
**Severity:** Critical

```python
# create() - Line 87-93
@api.model_create_multi
def create(self, vals_list):
    res = super(AccountMoveLine, self).create(vals_list if vals_list else [])
    for rec in res:
        if not rec.branch_id and rec.move_id and rec.move_id.branch_id:
            rec.branch_id = rec.move_id.branch_id.id  # triggers write() per record
    return res

# write() - Line 95-100
def write(self, vals):
    res = super(AccountMoveLine, self).write(vals)
    for rec in self:
        if not rec.branch_id and rec.move_id and rec.move_id.branch_id:
            rec.branch_id = rec.move_id.branch_id.id  # triggers ANOTHER write() per record
    return res
```

**The cascade effect:**
1. `invoice_sellout.create()` creates lines, then writes `basic_price` per record
2. Each per-record write triggers `multi_branches.write()`
3. `multi_branches.write()` calls `super().write()`, then writes `branch_id` per record
4. That branch_id write triggers another `write()` call

For an invoice with 50 lines, this creates approximately:
- 50 lines x 3 field writes (invoice_sellout) = 150 write calls
- Each of those 150 goes through multi_branches.write() = 150 more potential branch writes
- **Total: 300+ individual database write operations** instead of 1 batch create

**Recommendation for create():** Set `branch_id` in vals_list before calling super():

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        if not vals.get('branch_id') and vals.get('move_id'):
            move = self.env['account.move'].browse(vals['move_id'])
            if move.branch_id:
                vals['branch_id'] = move.branch_id.id
    return super().create(vals_list if vals_list else [])
```

**Recommendation for write():** Use batch update:

```python
def write(self, vals):
    res = super().write(vals)
    needs_branch = self.filtered(
        lambda r: not r.branch_id and r.move_id and r.move_id.branch_id
    )
    if needs_branch:
        for move_id in needs_branch.mapped('move_id'):
            lines = needs_branch.filtered(lambda r: r.move_id == move_id)
            lines.with_context(skip_branch_write=True).write({
                'branch_id': move_id.branch_id.id
            })
    return res
```

---

## Bottleneck #3: payslip_customization - SQL Per Analytic Key

**Impact:** 0.6% of total CPU (76 samples)
**Severity:** Medium
**Affected Operation:** Creating move lines with analytic distributions (payslips)

### Description

**File:** `payslip_customization/models/models.py:13-24`

```python
def create(self, values):
    for value in values:
        if value.get('analytic_distribution'):
            new_analytic = {}
            for key in value.get('analytic_distribution'):
                contracts = self.env['hr.contract'].search(
                    [('analytic_account_id', '=', key)], limit=1
                ).mapped('analytic_account_ids')  # SQL query per key per line
                if contracts:
                    for analytic in contracts:
                        new_analytic[analytic.id] = 100
            value['analytic_distribution'].update(new_analytic)
    return super().create(values)
```

For each move line value in the batch, and for each analytic distribution key, a separate `search()` query is executed against `hr.contract`. With payslips generating many lines with analytic distributions, this creates dozens of individual SQL queries.

**Recommendation:** Batch-collect all analytic IDs first, perform a single search, and build a lookup dictionary:

```python
def create(self, values):
    all_analytic_ids = set()
    for value in values:
        dist = value.get('analytic_distribution')
        if dist:
            all_analytic_ids.update(dist.keys())

    contracts_map = {}
    if all_analytic_ids:
        int_ids = [int(aid) for aid in all_analytic_ids]
        contracts = self.env['hr.contract'].search(
            [('analytic_account_id', 'in', int_ids)]
        )
        for contract in contracts:
            key = str(contract.analytic_account_id.id)
            contracts_map[key] = contract.analytic_account_ids

    for value in values:
        dist = value.get('analytic_distribution')
        if dist:
            new_analytic = {}
            for key in dist:
                if key in contracts_map:
                    for analytic in contracts_map[key]:
                        new_analytic[str(analytic.id)] = 100
            dist.update(new_analytic)

    return super().create(values)
```

---

## Additional Observations

### government_incentive/models/sale_order.py - order_create_invoices

**File:** `government_incentive/models/sale_order.py:38-110`
**Impact:** 18 samples (0.14%)

The `order_create_invoices` method creates invoices, then immediately creates payment registers and journal entries. It also calls `journal_entry.action_post()` (line 99), which triggers the full action_post chain described in Bottleneck #1.

The method also has a logic issue at line 69:
```python
if not incentive_journal_id or incentive_account_id:
```
This condition should likely be `if not incentive_journal_id or not incentive_account_id:` (missing `not`).

### _compute_name in multi_branches

**File:** `multi_branches/models/account_invoice.py:50-80`

The `_compute_name` method calls `next_by_id()` on sequences, which is a database operation. It depends on `branch_id` among other fields, meaning any change to branch_id triggers a recomputation and sequence consumption. This could lead to sequence gaps and unnecessary recomputation during the write cascade described in Bottleneck #2.

---

## Summary Table

| # | Module | File:Line | Issue | Impact | Severity | Fix Effort |
|---|--------|-----------|-------|--------|----------|------------|
| 1A | government_incentive | account_move.py:75 | Recursive action_post() | ~4-5% | Critical | Medium |
| 1B | restrict_invoicing + sale_returns_management | account_move.py:185, account_move.py:44 | Duplicate refund validation | ~2% | High | Medium |
| 1C | custom_discount_module | models.py:770-788 | Warning popup blocks every vendor bill | ~1% | Medium | Low |
| 1D | journal_audit + ghanim_custome_accounting | account_move.py:33, account_move.py:14 | Unnecessary overrides for all move types | ~0.5% | Low | Low |
| 2A | invoice_sellout | account_move_line.py:14-18 | 3 individual write() calls per line in create | ~3% | Critical | Low |
| 2B | multi_branches | account_invoice.py:87-100 | Per-record branch_id write in create() and write() | ~5.5% | Critical | Low |
| 3 | payslip_customization | models.py:18 | SQL query per analytic key per line | ~0.6% | Medium | Low |

**Estimated total CPU reduction after all fixes: 12-15%**

---

## Recommended Fix Priority

1. **Fix 2A + 2B first** (invoice_sellout + multi_branches batch writes) - Highest impact, lowest effort
2. **Fix 1A** (government_incentive recursive post) - High impact, medium effort
3. **Fix 1B** (consolidate duplicate refund validation) - Medium impact, medium effort
4. **Fix 3** (payslip batch contract search) - Low-medium impact, low effort
5. **Fix 1C** (discount warning optimization) - Low impact, low effort
6. **Fix 1D** (consolidate entry validations) - Low impact, low effort

---

*Report generated from flame graph profile captured on 2026-02-16 from Odoo.sh production instance.*
