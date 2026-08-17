# Company AI Rules for Gemini/Codex

**Version 2.0** | **Last Updated**: 2026-08-17

These instructions are centrally managed. Edit the canonical rules file instead of this generated file. 

---
title: Odoo Development Coding Standards
version: 1.0
last_updated: 2026-02-14
---

# Odoo Development Coding Standards

This document defines the mandatory coding standards and best practices for all Odoo development work. All developers must follow these guidelines.

## 1. Naming Conventions

### Module Names
- **MUST** start with `sh_` prefix
- Use lowercase with underscores
- Be descriptive and concise
- Example: `sh_custom_invoice`, `sh_inventory_management`

### File Names
- **MUST** start with `sh_` prefix
- Use lowercase with underscores
- Match the purpose of the file
- Example: `sh_sale_order.py`, `sh_invoice_report.xml`

### Model Names
- Use descriptive names following Odoo conventions
- Use dot notation for hierarchy: `sh.module.model`
- Example: `sh.custom.invoice`, `sh.inventory.transfer`

### Field Names
- Use snake_case
- Be descriptive and clear
- Avoid abbreviations unless commonly understood
- **Important:** `state`, `create_date`, `write_date`, `id`, `name` are **core Odoo fields** — DO use them in models. Never create custom fields with these same names as they will conflict with Odoo's system fields.
- Example: ✅ Add `state = fields.Selection(...)` for workflow, ❌ Don't create custom `my_state` field if standard `state` works

---

## 2. Security Requirements

### Access Rights
- **MUST** define proper access rights in `ir.model.access.csv`
- Every persistent model (non-abstract/transient) **MUST** have at least one ACL entry
- Scope `perm_unlink` tightly for sensitive records
- Never use `sudo()` unless absolutely necessary
- When `sudo()` is required:
  - Add a comment explaining why it's needed with business justification
  - Limit scope: use `sudo().browse(ids)` instead of broad queries
  - Document security implications clearly

### Record Rules (Row-Level Security)
- Define record rules for company-specific or user-scoped data
- Use proper domain syntax: `[('company_id', 'in', company_ids)]`
- Never hard-code company IDs; use dynamic company context
- Test with different user roles (Admin, Manager, User, Portal User)

### Controllers
- **NO OPEN CONTROLLERS** — All controllers must require authentication
- Use `@http.route(..., auth='user')` for authenticated endpoints
- Use `@http.route(..., auth='public')` only when explicitly required by business needs
- **NEVER** use `auth='none'` — this is a security vulnerability
- Validate all input parameters before processing
- Implement CSRF protection for state-changing operations using `csrf=True`

Example:
```python
# ❌ FORBIDDEN — Open controller
@http.route('/my/endpoint', auth='none')
def my_endpoint():
    pass

# ✅ CORRECT — Authenticated with CSRF
@http.route('/my/endpoint', auth='user', csrf=True, type='http')
def my_endpoint(self, **kw):
    pass
```

### SQL Injection Prevention
- **NEVER** use string formatting or concatenation for SQL queries
- Always use parameterized queries with positional placeholders
- Sanitize all user-provided identifiers (model names, field names)

Example:
```python
# ❌ FORBIDDEN — String formatting (SQL injection risk)
self.env.cr.execute("SELECT * FROM table WHERE id = %s" % record_id)

# ✅ CORRECT — Parameterized query
self.env.cr.execute("SELECT * FROM table WHERE id = %s", (record_id,))
```

---

## 3. Multi-Company Support

### When Required
- If the module handles company-specific data:
  - Sales, purchases, invoices, orders
  - Inventory, warehouse, stock movements
  - Accounting, journal entries, financial reports
  - Payroll, HR approvals, employee records
- **MUST** add `company_id` field to relevant models
- **MUST** add company-based record rules
- **MUST** test with multiple companies

### Implementation

**Model Field:**
```python
company_id = fields.Many2one(
    'res.company',
    string='Company',
    required=True,
    default=lambda self: self.env.company,
    ondelete='cascade'
)
```

**Record Rule (Allow multi-company + shared records):**
```xml
<record id="sh_model_company_rule" model="ir.rule">
    <field name="name">Multi-Company Access</field>
    <field name="model_id" ref="model_sh_custom_model"/>
    <field name="domain_force">[('company_id', 'in', company_ids)]</field>
</record>
```

**Domain Examples:**
- Restrict to current company: `[('company_id', '=', self.env.company.id)]`
- Allow shared + company-specific: `['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]`
- Never use: `[('company_id', '=', 1)]` — Always use dynamic context

**Multi-Company + Shared Records Pattern (Critical):**
```python
# ✅ CORRECT: Search with dynamic company context
def get_shared_and_company_records(self):
    """Return shared records + current company records."""
    company_ids = self.env.company.ids
    domain = ['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]
    return self.search(domain)

# ✅ Search all records including archived (for admin)
def get_all_with_shared(self):
    """Return all shared + company records (including archived)."""
    company_ids = self.env.company.ids
    domain = ['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]
    return self.with_context(active_test=False).search(domain)

# ✅ What to do when user switches companies
def on_company_switch(self, new_company_id):
    """Handle company context switch."""
    # Recalculate all company-dependent fields
    # Refresh record visibility based on new company
    # Clear any company-specific cache
    self.env.user.company_id = new_company_id
```

**Record Rule for Shared + Company-Specific Records:**
```xml
<!-- Allow access to shared records (company_id = False) AND user's companies -->
<record id="sh_model_shared_company_rule" model="ir.rule">
    <field name="name">Shared & Company Records</field>
    <field name="model_id" ref="model_sh_custom_model"/>
    <field name="domain_force">['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]</field>
</record>
```

**Computed Field Handling in Multi-Company:**
```python
class ShInvoice(models.Model):
    _name = 'sh.invoice'
    
    company_id = fields.Many2one('res.company', required=True)
    amount = fields.Float()
    
    @api.depends('amount', 'company_id')
    def _compute_tax_amount(self):
        """Compute tax based on company's tax settings."""
        for record in self:
            # Use record's company, not env.company
            tax_rate = record.company_id.tax_rate or 0
            record.tax_amount = record.amount * (tax_rate / 100)
    
    tax_amount = fields.Float(compute='_compute_tax_amount', store=True)
```

---

## 4. Code Structure

### Module Structure
```
sh_module_name/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── sh_*.py
├── views/
│   └── sh_*_views.xml
├── security/
│   ├── ir.model.access.csv
│   └── sh_*_security.xml
├── controllers/
│   ├── __init__.py
│   └── sh_*_controller.py
├── data/
│   └── sh_*.xml
├── demo/
│   └── sh_*_demo.xml
├── static/
│   └── src/
│       ├── js/
│       ├── css/
│       └── xml/
├── tests/
│   ├── __init__.py
│   └── test_*.py
└── README.md
```

### Manifest & Metadata
- Every `__manifest__.py` **MUST** declare:
  - `name`: User-friendly module name
  - `version`: Format as `{odoo_version}.{major}.{minor}.{patch}` (e.g., `17.0.1.2.0`)
  - `author`: Company or developer name
  - `license`: e.g., `LGPL-3`, `AGPL-3`
  - `category`: e.g., `Accounting`, `Inventory`
  - `depends`: List all module dependencies (check spelling and order)
  - `description`: Clear description of functionality
  - `installable`: `True` for installable modules
  - `application`: `True` only for standalone apps with root menus
  - `auto_install`: `False` unless bridging established dependencies

**Data File Load Order in Manifest:**
1. Security (`security/ir.model.access.csv`)
2. Record Rules (`security/sh_*_security.xml`)
3. Views and Menus (`views/sh_*_views.xml`)
4. Business Data (`data/sh_*.xml`)
5. Reports (`reports/sh_*_report.xml`)
6. Demo Data (`demo/sh_*_demo.xml`)

### Tests and Demo Data
- Keep demo data **only** inside `demo/` folder
- Production data belongs in `data/` — **never mix them**
- Write unit tests for critical workflows
- Tests are optional but recommended for complex business logic
- Test folder structure: `tests/test_model_name.py`

### Python Code Standards
- Follow **PEP 8** strictly
- Maximum line length: **120 characters**
- Use meaningful variable names (avoid single letters except `i`, `k`, `v`)
- Add docstrings to all classes and public methods:
  ```python
  def process_invoice(self, invoice_id):
      """Process invoice and generate accounting entries."""
      pass
  ```
- Use type hints where applicable (Python 3.7+)
- No trailing whitespace
- Proper imports: stdlib → third-party → local

---

## 5. Security Best Practices

### Input Validation
- Validate all user inputs at system boundaries (controllers, imports, APIs)
- Use Odoo's built-in validation mechanisms
- Implement `@api.constrains` for field-level validations
- Reject invalid data types, ranges, and lengths before processing

### XSS Prevention
- Sanitize HTML content using `tools.html_sanitize()`
- Use `t-esc` for user-generated content in templates (default, safe)
- Avoid `t-raw` unless absolutely necessary for trusted HTML
- Document why `t-raw` is needed if used

### Sensitive Data
- Never log sensitive information (passwords, tokens, credit cards, PAN)
- Use appropriate field types:
  - `fields.Char(password=True)` for passwords
  - Encrypt sensitive data before storing
- Use `ir.config_parameter` for API keys (with encryption)
- Clear sensitive data from logs and error messages

### ORM Safety
- Call `ensure_one()` before code assuming a single record
- Avoid accessing `self.id` or `self.name` without checking context
- Never rely on deprecated `@api.one` or `@api.multi` decorators
- Use correct decorators for their scope:
  - `@api.model_create_multi` for batch create support
  - `@api.depends` for computed field dependencies
  - `@api.constrains` for field validations
  - `@api.onchange` for user feedback in forms

### Exception Handling
- Use Odoo-specific exceptions with translatable messages:
  ```python
  from odoo.exceptions import UserError, ValidationError, AccessError
  
  raise ValidationError(_("Invalid value: %s") % value)
  raise UserError(_("Please configure settings first"))
  ```
- Avoid raw `Exception` unless re-raising after logging
- Provide clear, actionable error messages to users

---

## 6. Performance Considerations

### Database Queries
- **Avoid N+1 queries**: Use `search_read()` instead of looping with `search()` + `read()`
- Prefetch related records: `mapped()`, `filtered()`, `with_prefetch()`
- Implement proper indexing on frequently searched fields: `index=True`
- Use raw SQL only for complex aggregations (with `@api.depends_context('force_company')`)
- Limit search results: `search(..., limit=500)` to prevent memory issues

### Computed Fields
- Use `store=True` only when the field is frequently displayed (trade-off: writes slower)
- Implement proper `@api.depends()` to trigger recalculation
- Avoid heavy computations (complex loops, external API calls) in computed fields
- Use `compute` for display-only, non-stored fields

### Cron Jobs / Scheduled Actions
- Process records in **batches** (e.g., 100–500 records per batch)
- Use pagination with `limit` and `offset` for large datasets
- **Never** process all records in a single transaction
- Explicit commit after each batch: `self.env.cr.commit()`
- Handle errors gracefully — one record failure must NOT stop the entire job
- Log failed records with `_logger.exception()`
- Ensure the job handles **10K+ records** without timeout

Example:
```python
import logging
_logger = logging.getLogger(__name__)

def _cron_process_records(self):
    batch_size = 100
    offset = 0
    
    while True:
        records = self.search([], limit=batch_size, offset=offset)
        if not records:
            break
            
        for record in records:
            try:
                record.process()
            except Exception as e:
                _logger.exception("Failed to process record %s", record.id)
                continue
        
        self.env.cr.commit()
        offset += batch_size
```

---

## 7. Archive Flow Implementation

### Overview
Use **archive (soft delete)** instead of permanent delete for business-critical data. Archive is the standard Odoo pattern for master data management.

### Model Implementation

**Add active field:**
```python
class ShCustomModel(models.Model):
    _name = 'sh.custom.model'
    _description = 'Custom Model'
    
    name = fields.Char(required=True)
    active = fields.Boolean(
        default=True,
        help="Uncheck to archive this record"
    )
```

**Archive record - Implementation (REQUIRED):**
```python
# CRITICAL: These action methods MUST be implemented
def action_archive(self):
    """Archive this record and related child records."""
    # Validate before archiving
    if self.state not in ['draft', 'confirmed']:
        raise ValidationError(_("Cannot archive records in %s state") % self.state)
    
    # Archive related child records
    if self.child_ids:
        self.child_ids.action_archive()
    
    # Archive this record
    self.write({'active': False})
    
    # Log activity
    self.message_post(body=_("Record archived"))
    
    return {
        'type': 'ir.actions.client',
        'tag': 'display_notification',
        'params': {
            'title': _('Success'),
            'message': _('Record archived successfully'),
            'type': 'success',
        }
    }

def action_unarchive(self):
    """Restore archived record."""
    # Check if parent is archived (cannot unarchive child of archived parent)
    if self.parent_id and not self.parent_id.active:
        raise ValidationError(_("Cannot restore record. Parent record is archived."))
    
    self.write({'active': True})
    self.message_post(body=_("Record restored"))
    
    return {
        'type': 'ir.actions.client',
        'tag': 'display_notification',
        'params': {
            'title': _('Success'),
            'message': _('Record restored successfully'),
            'type': 'success',
        }
    }

# Method 1: Direct write (basic)
record.active = False

# Method 2: Bulk archive
records.write({'active': False})

# Method 3: From form view button (recommended - with validation)
# Use action_archive() method shown above
```

### View Implementation

**Form View:**
```xml
<form>
    <header>
        <button name="action_archive" type="object" string="Archive"
                class="btn-warning" 
                attrs="{'invisible': [('active', '=', False)]}"/>
        <button name="action_unarchive" type="object" string="Restore"
                class="btn-info" 
                attrs="{'invisible': [('active', '=', True)]}"/>
    </header>
    <sheet>
        <field name="name"/>
        <!-- other fields -->
    </sheet>
</form>
```

**List View:**
```xml
<list>
    <field name="name"/>
    <field name="active" widget="boolean_toggle"/>
    <!-- Apply default filter to hide archived -->
</list>
```

**Search/Filter:**
```xml
<search>
    <filter name="archived" string="Archived"
            domain="[('active', '=', False)]"/>
    <filter name="active" string="Active"
            domain="[('active', '=', True)]"/>
</search>
```

### Business Logic Integration

**Exclude archived in searches (default):**
```python
# Instead of bare search
records = self.search([('name', '=', 'Test')])

# Use active domain filter
records = self.search([('active', '=', True), ('name', '=', 'Test')])
```

**Force include archived when needed:**
```python
# Search active + archived
all_records = self.with_context(active_test=False).search([('name', '=', 'Test')])
```

**Prevent operations on archived records:**
```python
@api.constrains('active')
def _check_archived_operations(self):
    for record in self:
        if not record.active and record.state != 'draft':
            raise ValidationError(_("Cannot archive %s in %s state") % (record.name, record.state))
```

**Cascade archive related records:**
```python
class ShCustomModel(models.Model):
    _name = 'sh.custom.model'
    
    parent_id = fields.Many2one('sh.custom.model', ondelete='cascade')
    child_ids = fields.One2many('sh.custom.model', 'parent_id')
    
    def write(self, values):
        result = super().write(values)
        
        # Archive children when parent is archived
        if 'active' in values and not values['active']:
            self.child_ids.write({'active': False})
        
        return result
```

### Record Rules with Archive

```xml
<record id="sh_model_archive_rule" model="ir.rule">
    <field name="name">Active Records Only</field>
    <field name="model_id" ref="model_sh_custom_model"/>
    <field name="domain_force">[('active', '=', True)]</field>
</record>
```

---

## 8. Documentation Requirements

### Module Documentation
- **README.md**: Include module description, features, configuration steps, and usage examples
- **Changelog**: Document version updates with date and description
- **Configuration Instructions**: How to set up features, required settings, etc.

### Code Comments
- Document **why**, not what — code should be self-explanatory
- Add TODO comments with name and date: `# TODO: Improve performance (John - 2026-08-12)`
- Comment complex business logic and non-obvious workarounds
- Remove obsolete comments during refactoring

### Commit Messages
- Format: `[MODULE_NAME] Concise description`
- Example: `[sh_custom_invoice] Add archive flow for invoices`
- Include what changed and why, not just "Fix bug" or "Update"
- One logical change per commit

---

## 9. Testing Requirements

### Mandatory Tests
- Write unit tests for critical business logic
- Test all user roles: Admin, Manager, User, Portal User
- Test multi-company scenarios (if applicable)
- Test archive/restore behavior
- Test edge cases: empty data, boundary values, zero amounts
- Test error handling and validation

### Testing Checklist
```python
class TestShCustomModel(TransactionCase):
    def setUp(self):
        super().setUp()
        self.model = self.env['sh.custom.model']
    
    def test_create_valid_record(self):
        """Test creating a valid record."""
        record = self.model.create({'name': 'Test'})
        self.assertTrue(record.active)
    
    def test_archive_record(self):
        """Test archiving a record."""
        record = self.model.create({'name': 'Test'})
        record.active = False
        self.assertFalse(record.active)
    
    def test_search_excludes_archived(self):
        """Test that search excludes archived records by default."""
        self.model.create({'name': 'Active', 'active': True})
        self.model.create({'name': 'Archived', 'active': False})
        
        results = self.model.search([])
        self.assertEqual(len(results), 1)
        self.assertEqual(results.name, 'Active')
```

---

## 10. File Header Standards

### Mandatory Header
Every Python file (`.py`) **MUST** start with this exact header:

```python
# -*- coding: utf-8 -*-
# Copyright (C) Softhealer Technologies Pvt. Ltd.
```

### Rules
- Must be the **first two lines** of every `.py` file
- Applies to ALL Python files: models, wizards, controllers, `__init__.py`, `__manifest__.py`
- Exact format required — no variations or abbreviations

---

## 11. Code Review Standards

### Comparative Analysis Methodology
When reviewing custom code, **compare directly with standard Odoo addons** instead of relying solely on predefined rules.

### Review Process
1. **Extract custom code**: All models, methods, fields, views, and security rules
2. **Find standard implementations**: Search community/enterprise addons for similar functionality
3. **Component comparison**:
   - **Models**: Inheritance patterns, field definitions, method implementations, decorators
   - **Views**: XML structure, xpath usage, widget placement, visibility rules
   - **Security**: Access rights group assignments, record rule domains
4. **Detect deprecated patterns**: If custom code uses a pattern NOT in any standard module, flag as potentially deprecated
5. **Performance pattern check**: Compare batch operations, search patterns, and loop structures
6. **Override tracing**: For every overridden method:
   - Trace to core definition
   - Confirm `super()` is called
   - Document the resulting MRO chain

### Finding Output Format
```
📁 Custom: [file_path:line_number]
🔗 Standard Reference: [standard_module/file_path:line_number]

❌ ISSUE: [What's different/wrong]
✓ STANDARD PATTERN: [How standard code does it]
💡 RECOMMENDATION: [Specific fix with code example]
⚠️ IMPACT: [Why this matters — performance/security/maintainability]
```

**Key Rule**: Every recommendation **MUST** include a reference to actual standard Odoo code.

---

## 12. Integration & Compatibility

### Standard Addon Compatibility
- Compare custom methods with similar standard addon implementations
- Verify no conflicts with core Odoo functionality
- Override methods **must** call `super()` at the correct point
- Field additions must not break existing views or inheritance chains
- Computed fields must not interfere with standard computations

### Cross-Module Dependencies
- Review dependencies in `__manifest__.py` for completeness and correctness
- Check for circular dependencies (A depends on B, B depends on A)
- Validate compatibility with other custom modules in the project
- Test inheritance chains for conflicts and proper MRO
- Verify **no naming collisions**:
  - Models with same name
  - Fields with same name on inherited models
  - Views with same XML IDs
  - Menu items with duplicate IDs

### Database Compatibility
- Use PostgreSQL features appropriately (no MySQL-specific syntax)
- Write migration scripts if schema changes required
- Test upgrades on databases with existing records
- Use `ALTER TABLE` safely with proper constraints

---

## 13. Business Logic Validation

### Workflow State Transitions
- Control state transitions with explicit methods (not direct writes)
- Required approvals/validations must be enforced before state change
- Draft, Confirm, Cancel operations must be handled correctly
- Workflow sequence must be logically sound (e.g., can't go from "Cancelled" to "Confirmed")

Example:
```python
def action_confirm(self):
    """Move from draft to confirmed state."""
    if self.state != 'draft':
        raise UserError(_("Only draft records can be confirmed"))
    self.state = 'confirmed'

def action_cancel(self):
    """Move to cancelled state."""
    if self.state not in ['draft', 'confirmed']:
        raise UserError(_("Cannot cancel record in %s state") % self.state)
    self.state = 'cancelled'
```

### Data Integrity
- Constraints properly defined using `@api.constrains` and SQL constraints
- Onchange methods provide proper user feedback (no silent failures)
- Cascade delete (`ondelete='cascade'`) only for child records
- Set delete (`ondelete='set null'`) for optional relationships
- Restrict delete (`ondelete='restrict'`) for critical links
- Referential integrity must be maintained at all times

Example:
```python
class ShInvoiceLine(models.Model):
    _name = 'sh.invoice.line'
    
    invoice_id = fields.Many2one(
        'sh.invoice',
        required=True,
        ondelete='cascade'  # Delete lines when invoice deleted
    )
    product_id = fields.Many2one(
        'product.product',
        ondelete='restrict'  # Prevent product deletion if used
    )
```

---

## 14. Client Expected Common Points (UX & Business Requirements)

### Field Labels & Configuration
- Use meaningful, user-friendly labels for all new fields
- Essential fields **MUST** have clear, helpful tooltips
- Avoid technical jargon in UI labels

### Warnings & Notifications
- Warnings must identify **exact missing information**
- Provide meaningful guidance: "Please configure X in Y settings"
- Follow Odoo warning type definitions (`warning`, `title`, `message`)
- Use success notifications to confirm actions: "✓ 50 invoices processed"

### Tracking & Communication
- Important business fields **MUST** have `tracking=True`
- Every new business model should inherit:
  - `mail.thread` (Chatter support)
  - `mail.activity.mixin` (Activities for follow-ups)

Example:
```python
class ShInvoice(models.Model):
    _name = 'sh.invoice'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    
    amount = fields.Float(tracking=True)  # Track amount changes in chatter
    state = fields.Selection(tracking=True)
```

### Search & Filter Functionality
- New custom fields **MUST** be available in Search (Filter/Group By) where applicable
- Filter/Group By labels must match field labels exactly
- Implement meaningful filter presets (Active, Archived, Overdue, etc.)

### Display Standards
- **Monetary values**: Show currency symbols in all views
- **Calculated values with missing inputs**: Display `N/A`, not misleading defaults
- **Form views**: Provide complete information hierarchy (sections, tabs)
- **List views**: Custom fields should be optional/hideable where practical

### View Coverage
New models should provide complete analytics-ready views:
- Form view (full record editing)
- List view (record overview)
- Search view (filters, sorting)
- Pivot view (data analysis)
- Graph view (visual trends)

### Smart Buttons
- Hide smart buttons when count is `0`
- Open record directly when count is `1`
- Show meaningful counts for batch actions

Example:
```xml
<button class="oe_stat_button" name="action_open_related"
        type="object" icon="fa-folder-open"
        attrs="{'invisible': [('related_count', '=', 0)]}">
    <field name="related_count" widget="statinfo"/>
</button>
```

### Mass Actions
- Remove obsolete mass actions during refactoring
- Use meaningful action names: ❌ `Action1`, ✅ `Update Material Code`
- Confirm destructive actions (archive, state change)

### UX & Portal
- Portal links **MUST** use access tokens where required
- Portal records **MUST** remain strictly customer-scoped
- Validate customer can only access their own records

### Localization
- Translate all **customer/supplier-facing** text
- Validate locale-specific number/date formatting (including German `DD.MM.YYYY`)
- Test with multiple languages during development

### Reports & Documents
- Keep UI/PDF alignment consistent across flows
- Use consistent file naming: `{document_type}_{date}_{number}.pdf`
- Email templates must have clean layout and correct business content

### Module Presentation
- Include a proper app icon for new modules (`static/src/img/icon.png`)
- Create meaningful README with examples
- Document configuration and usage

### Data Management
- Add cleanup/garbage-collection strategy for temporary/obsolete data
- Implement data archiving for historical records
- Remove old demo data after testing

### Key Business Fields
- Ensure key business fields are available in filter functionality
- Only exclude fields explicitly mentioned in requirements
- Document why fields are excluded

---

## 15. Essential Technical Validation Checklist

**Apply these checks before completing ANY Odoo development.**

### Fields & Models
- ✅ Every field in Python, XML, domains, contexts, reports, JS, `@api.depends`, `@api.constrains`, `@api.onchange` exists on target model
- ✅ Every model and inherited model exists; providing module declared in `depends`
- ✅ Field types correct: comodel, inverse field, related path, selection values, currency field, `ondelete`, `compute`, `inverse`, `store`
- ✅ Compute methods assign their field for **every record** and **every condition**
- ✅ Avoid dotted field names in `@api.constrains` (Odoo ignores them)
- ✅ `selection_add` provides required `ondelete` fallback
- ✅ `copy=False` set for: state, approval data, sequence/reference values, tokens, signatures, completion dates

### Input & Value Validation
- ✅ Validate required values, data types, allowed selections, numeric ranges, date ranges, string length, file types
- ✅ Treat `False`, `None`, `0`, empty strings, empty recordsets, missing dict keys as separate cases
- ✅ Validate user-provided record IDs with `browse(ids).exists()`
- ✅ Verify access rights, record rules, ownership, company before using records
- ✅ **Never trust** IDs, domains, context values, filenames, URLs, SQL identifiers, model names from controllers/APIs/webhooks
- ✅ Use Odoo date/datetime/float helpers, never unsafe conversions
- ✅ Validate relational commands; verify related records belong to allowed company
- ✅ Imports/APIs must not depend on form `onchange`; enforce required values in model logic

### XML, Views, Domains, External IDs
- ✅ Every `ref`, action, menu, view, group, report, template, external ID exists before use
- ✅ Manifest file load order: security → views → data → reports → demo
- ✅ Every field in view belongs to view's model; fields in modifiers available in that view
- ✅ Domains validated against correct model, field type, operator, value type
- ✅ Use Odoo 19 syntax: `<list>` for list views, inline `invisible`, `readonly`, `required` expressions
- ✅ Inherited-view `xpath` matches element in installed parent view
- ✅ Module upgrade detects invalid fields, XML syntax, missing IDs, broken xpaths, invalid expressions

### Python Methods & ORM
- ✅ Overridden method signature matches parent; return type correct
- ✅ Use `ensure_one()` only for singleton methods; iterate for full recordsets
- ✅ Preserve batch behavior in `create()` with `@api.model_create_multi`
- ✅ Call `super()` correctly; verify complete inheritance chain
- ✅ Check dict access before using `values['key']` when optional; use `.get()` when missing is acceptable
- ✅ Never pass unsaved/NewId record where database ID required
- ✅ After raw SQL writes, synchronize ORM cache

### Minimum Error-Prevention Test
Before completing any feature:

1. ✅ Install module on clean test database → no errors
2. ✅ Upgrade on database with existing records → no errors
3. ✅ Create record with minimum valid input → works
4. ✅ Try invalid, missing, empty, zero, boundary input → proper errors
5. ✅ Test as normal user (not admin) → works with restrictions
6. ✅ Test duplicate/archive behavior → works correctly
7. ✅ Test with multiple companies (if applicable) → works for each
8. ✅ Check server traceback and browser console → no errors
9. ✅ Add regression test for every confirmed technical error (when practical)

**If error mentions undefined field, missing model, or missing external ID**: First verify technical name, target model, manifest dependency, file load order, and module upgrade status before changing logic.

---

## 16. Code Search & Problem-Solving Process

### When Code Errors Occur
1. **Build confidence with standard reference**:
   - Search for similar implementations in standard Odoo addons
   - Compare your pattern with proven Odoo modules
   - Document the reference for the fix

2. **Repeated failures**:
   - If you've tried multiple times without success
   - Read standard relevant Odoo code from the workspace
   - If standard addons not found, ask the user for standard Odoo module paths to save time/tokens

3. **Documentation approach**:
   - Always cite the standard Odoo pattern you found
   - Include line numbers and module names
   - Explain why the standard pattern solves your issue

---

## 17. File Load Order & Manifest Sequences

### Critical for Avoiding "External ID Not Found" Errors

The order in `__manifest__.py` `data` list matters. Files load sequentially; an external ID must load before another file references it.

**Correct Manifest Load Order:**
```python
'data': [
    # 1. Security (required before everything else)
    'security/ir.model.access.csv',
    'security/sh_*_security.xml',  # Record rules
    
    # 2. Views & Menus (reference security groups)
    'views/sh_*_views.xml',
    'views/sh_*_menu.xml',
    
    # 3. Business Data (references views, uses record IDs)
    'data/sh_*_data.xml',
    'data/sh_*_sequences.xml',
    'data/sh_*_cron.xml',
    
    # 4. Reports (references data)
    'reports/sh_*_report.xml',
],
'demo': [
    # 5. Demo Data (last, for testing only)
    'demo/sh_*_demo.xml',
],
```

**Why This Order:**
- ACL and record rules must exist before views that reference groups
- External IDs in views must exist before data references them
- Demo data loads last, after everything else

**Common Error Prevention:**
- ❌ `External ID "ir.group_accounting" not found` → ACL file not in `data` list
- ❌ `Key error: 'sh.custom.model'` → Model not defined in `depends`
- ❌ `Xpath cannot find element` → Parent view defined in different module, dependency missing

---

## 18. Manifest File Standards

### `__manifest__.py` Template
```python
# -*- coding: utf-8 -*-
# Copyright (C) Softhealer Technologies Pvt. Ltd.

{
    'name': 'Module Display Name',
    'version': '17.0.1.0.0',
    'category': 'Accounting',
    'author': 'Softhealer Technologies',
    'license': 'LGPL-3',
    'depends': [
        'base',
        'sale',
        'account',
    ],
    'data': [
        'security/ir.model.access.csv',
        'security/sh_module_security.xml',
        'views/sh_module_views.xml',
        'data/sh_module_data.xml',
        'reports/sh_module_report.xml',
    ],
    'demo': [
        'demo/sh_module_demo.xml',
    ],
    'external_dependencies': {
        'python': ['requests', 'openpyxl'],
    },
    'installable': True,
    'application': False,
    'auto_install': False,
    'description': '''
        Brief description of module.
        
        Features:
        - Feature 1
        - Feature 2
    ''',
}
```

---

## 19. PDF Report Design & Dynamic Values

### Overview
PDF reports must handle dynamic, variable-length content gracefully. Design for both minimum and maximum realistic data values.

### Dynamic Value Handling

**Problem**: Reports may display different amounts, text lengths, and quantities that vary significantly between records.

**Solution**: Design templates with flexible spacing and conditional sections.

### Implementation Guidelines

**1. Variable Amount/Monetary Values**
```xml
<t t-set="formatted_amount" t-value="'{:,.2f}'.format(o.total_amount)"/>
<t t-if="o.total_amount > 999999">
    <!-- High-value styling (larger font, bold) -->
    <span class="high-value" t-esc="formatted_amount"/>
</t>
<t t-else="">
    <span class="normal-value" t-esc="formatted_amount"/>
</t>
```

**2. Handling Long Text Fields**
```xml
<!-- Limit text with word wrapping -->
<div class="description" style="word-break: break-word; max-width: 100%;">
    <t t-esc="o.description"/>
</div>

<!-- For very long content, use model method (NOT inline Python) -->
<t t-set="truncated_text" t-value="o._truncate_text(o.notes, 100)"/>
<span t-esc="truncated_text"/>
```

**Python Model Method for Text Truncation:**
```python
@api.model
def _truncate_text(self, text, max_length=100, suffix='...'):
    """Truncate text to max length with suffix if needed."""
    if not text:
        return text
    text_str = str(text)
    if len(text_str) > max_length:
        return text_str[:max_length - len(suffix)] + suffix
    return text_str
```

**3. Dynamic Line Items with Variable Quantities**
```xml
<!-- Handle tables that grow/shrink -->
<table style="width: 100%; page-break-inside: avoid;">
    <tbody>
        <tr t-foreach="o.order_line" t-as="line">
            <td t-esc="line.quantity"/>
            <td t-esc="line.product_id.name"/>
            <td class="text-right" t-esc="'{:,.2f}'.format(line.price_subtotal)"/>
        </tr>
    </tbody>
</table>
```

**4. Conditional Sections for Large Values**
```xml
<!-- Hide sections if empty; adjust spacing if large -->
<t t-if="o.notes">
    <div class="section-notes">
        <h4>Notes</h4>
        <p t-esc="o.notes"/>
    </div>
</t>

<!-- Dynamic spacing for variable content height -->
<t t-set="has_discount" t-value="sum(o.order_line.mapped('discount')) > 0"/>
<t t-if="has_discount">
    <div class="discount-section">
        <p>Discount Applied: <t t-esc="sum(o.order_line.mapped('discount'))"/>%</p>
    </div>
</t>
```

### CSS Best Practices for Variable Content

```css
/* Prevent overflow of large numbers */
.amount {
    font-family: monospace;
    text-align: right;
    word-break: break-word;
}

/* Flexible table layouts */
table {
    width: 100%;
    table-layout: auto;
    border-collapse: collapse;
}

td {
    padding: 8px;
    border: 1px solid #ddd;
    word-wrap: break-word;
    word-break: break-word;
}

/* Prevent page breaks in middle of important content */
.no-break {
    page-break-inside: avoid;
}

/* Handle high-value displays */
.high-value {
    font-size: 18px;
    font-weight: bold;
    color: #d63031;
}
```

### Value Formatting Examples

**Python Model Method for Report:**
```python
@api.model
def _format_currency(self, amount, currency=None):
    """Format amount with proper currency symbol and decimals."""
    if not currency:
        currency = self.env.company.currency_id
    
    formatted = "{:,.2f}".format(float(amount))
    return f"{currency.symbol} {formatted}"

@api.model
def _format_large_amount(self, amount):
    """Format very large amounts (e.g., in millions)."""
    if abs(amount) >= 1000000:
        return "{:,.1f}M".format(amount / 1000000)
    elif abs(amount) >= 1000:
        return "{:,.0f}K".format(amount / 1000)
    else:
        return "{:,.2f}".format(amount)
```

**XML Template Usage:**
```xml
<t t-set="formatted" t-value="o._format_currency(o.total_amount)"/>
<p class="amount" t-esc="formatted"/>

<!-- For large amounts with special formatting -->
<t t-set="large_fmt" t-value="o._format_large_amount(o.revenue)"/>
<p class="high-value" t-esc="large_fmt"/>
```

### Testing Dynamic Reports

Before finalizing reports, test with:
1. ✅ **Minimum values**: Zero amounts, single-line items, empty optional fields
2. ✅ **Maximum values**: Large numbers (999,999.99), very high amounts (millions)
3. ✅ **Variable lengths**: Long text (>200 chars), multiple line items (50+), many optional sections
4. ✅ **Page breaks**: Ensure content doesn't orphan across pages
5. ✅ **Alignment**: Verify numbers align right, text flows left
6. ✅ **Localization**: Test currency symbols, date formats in different locales

### Common Report Issues & Solutions

| Issue | Solution |
|-------|----------|
| Numbers overflow column | Use `monospace` font, `text-align: right`, `word-break: break-word` |
| Long product names break layout | Set `max-width`, use `word-wrap: break-word` |
| Table grows beyond page | Use `page-break-inside: avoid` on rows, `limit` on line items |
| Currency symbols misaligned | Use fixed-width columns or table with separate `<td>` for symbol |
| High amounts look cramped | Increase font size conditionally based on amount threshold |
| Dates format incorrectly in PDF | Use Odoo `format_date()` helper, not string formatting |

---

## 20. Field Visibility & Access-Based Controls

### Overview
Fields can be automatically hidden, made read-only, or shown based on user access groups defined in `ir.model.access.csv`.

### Automatic Field Hiding Based on Access Rights

When you define access rights for a model in `ir.model.access.csv`, Odoo can automatically control field visibility in XML views.

**Key Concept**: If a user does NOT have `perm_read` (read permission) on a model, fields can be conditionally hidden in views.

### Implementation

**1. Define Access Groups in `security/ir.model.access.csv`**
```csv
id,name,model_id:id,group_id:id,perm_create,perm_read,perm_write,perm_unlink
sh_invoice_user,Invoice User,model_sh_invoice,base.group_user,True,True,True,False
sh_invoice_manager,Invoice Manager,model_sh_invoice,base.group_erp_manager,True,True,True,True
sh_invoice_accountant,Invoice Accountant,model_sh_invoice,account.group_account_user,True,True,True,False
sh_invoice_financial,Invoice Financial,model_sh_invoice,account.group_account_manager,True,True,True,True
```

**2. Hide Fields in XML Based on Group**

**Method A: Using `groups` attribute (Direct Group Check)**
```xml
<field name="cost_price" groups="sh_invoice_module.sh_invoice_manager"/>
<!-- Field only visible to "sh_invoice_manager" group -->

<field name="profit_margin" groups="account.group_account_manager"/>
<!-- Multiple groups: visible to any user in these groups -->
```

**Method B: Using `attrs` with invisible (More Flexible)**
```xml
<!-- Hide field if user NOT in specific group -->
<field name="internal_notes" 
        attrs="{'invisible': [('__context__', '!=', 'readonly')]}"/>

<!-- Conditional visibility based on both group AND record state -->
<field name="approval_amount"
        attrs="{'invisible': [('state', '!=', 'draft')],
                'readonly': [('approved', '=', True)]}"/>
```

**Method C: Python-Based Access Check (Most Reliable)**
```xml
<!-- In form view, use context to determine visibility -->
<field name="financial_data" 
        attrs="{'invisible': [('user_can_see_financials', '=', False)]}"/>
```

**Python Model:**
```python
class ShInvoice(models.Model):
    _name = 'sh.invoice'
    
    cost_price = fields.Float(string='Cost Price')
    
    @api.depends_context('uid')
    @api.depends('state')
    def _compute_user_can_see_financials(self):
        """Check if user has financial access group."""
        for record in self:
            has_access = self.env.user.has_group('account.group_account_manager')
            record.user_can_see_financials = has_access
    
    user_can_see_financials = fields.Boolean(
        compute='_compute_user_can_see_financials',
        store=False
    )
```

### Complete Example: Multi-Level Access Control

**Model Definition:**
```python
class ShCustomModel(models.Model):
    _name = 'sh.custom.model'
    
    name = fields.Char(required=True)
    description = fields.Text()
    
    # Standard fields - visible to all
    amount = fields.Float()
    
    # Manager-only fields
    cost_price = fields.Float(
        string='Cost Price (Manager Only)',
        help="Internal cost - only visible to managers"
    )
    discount_percentage = fields.Float()
    
    # Accountant-only fields
    accounting_code = fields.Char()
    journal_entry_id = fields.Many2one('account.move')
    
    # Financial analysis fields - requires account manager
    profit_margin = fields.Float(compute='_compute_profit_margin')
    roi = fields.Float(compute='_compute_roi')
    
    # Computed fields for access control
    can_view_cost = fields.Boolean(compute='_compute_can_view_cost')
    can_view_accounting = fields.Boolean(compute='_compute_can_view_accounting')
    
    @api.depends_context('uid')
    def _compute_can_view_cost(self):
        """User can view cost if in manager group."""
        has_access = self.env.user.has_group('base.group_erp_manager')
        for record in self:
            record.can_view_cost = has_access
    
    @api.depends_context('uid')
    def _compute_can_view_accounting(self):
        """User can view accounting if in account manager group."""
        has_access = self.env.user.has_group('account.group_account_manager')
        for record in self:
            record.can_view_accounting = has_access
    
    def _compute_profit_margin(self):
        """Compute profit margin (only accessible to managers)."""
        for record in self:
            if record.amount and record.cost_price:
                record.profit_margin = ((record.amount - record.cost_price) / record.amount) * 100
            else:
                record.profit_margin = 0
    
    def _compute_roi(self):
        """Compute ROI (only accessible to account managers)."""
        for record in self:
            if record.cost_price:
                record.roi = ((record.amount - record.cost_price) / record.cost_price) * 100
            else:
                record.roi = 0

    def _protect_sensitive_compute(self):
        """CRITICAL: Prevent unauthorized access to sensitive computed fields."""
        if not self.env.user.has_group('account.group_account_manager'):
            # Return empty/masked values for unauthorized users
            self.cost_price = 0
            self.profit_margin = 0
            self.roi = 0
```

**Protected Compute Method Pattern:**
```python
@api.depends('amount', 'cost_price')
def _compute_profit_margin(self):
    """Compute profit margin with security check."""
    for record in self:
        # Check authorization INSIDE compute method (not just UI)
        if not self.env.user.has_group('base.group_erp_manager'):
            record.profit_margin = 0  # Mask from unauthorized users
            continue
        
        # Only compute if authorized
        if record.amount and record.cost_price:
            record.profit_margin = ((record.amount - record.cost_price) / record.amount) * 100
        else:
            record.profit_margin = 0
```

**API Endpoint Field Filtering (Controller Pattern):**
```python
from odoo import http
from odoo.exceptions import AccessError

class ShInvoiceController(http.Controller):
    @http.route('/api/invoice/<int:invoice_id>', type='json', auth='user')
    def get_invoice(self, invoice_id):
        """Get invoice data with field-level filtering."""
        invoice = self.env['sh.invoice'].browse(invoice_id)
        
        # Base fields accessible to all authenticated users
        data = {
            'id': invoice.id,
            'number': invoice.number,
            'amount': invoice.amount,
        }
        
        # Sensitive fields - only for authorized users
        if self.env.user.has_group('base.group_erp_manager'):
            data['cost_price'] = invoice.cost_price
            data['profit_margin'] = invoice.profit_margin
        
        if self.env.user.has_group('account.group_account_manager'):
            data['accounting_code'] = invoice.accounting_code
            data['roi'] = invoice.roi
        
        return data
```

**XML View with Access Controls:**
```xml
<form>
    <header>
        <!-- Standard buttons visible to all -->
        <button name="action_confirm" type="object" string="Confirm" states="draft"/>
    </header>
    <sheet>
        <!-- Standard information - visible to all -->
        <group>
            <field name="name"/>
            <field name="description"/>
            <field name="amount"/>
        </group>
        
        <!-- Manager-only section -->
        <group string="Manager Information" 
               attrs="{'invisible': [('can_view_cost', '=', False)]}">
            <field name="cost_price"/>
            <field name="discount_percentage"/>
            <field name="profit_margin"/>
        </group>
        
        <!-- Accountant-only section -->
        <group string="Accounting Details"
               attrs="{'invisible': [('can_view_accounting', '=', False)]}">
            <field name="accounting_code"/>
            <field name="journal_entry_id"/>
            <field name="roi"/>
        </group>
        
        <!-- Hidden fields (no display but available to authorized users) -->
        <field name="cost_price" invisible="1" groups="base.group_erp_manager"/>
        <field name="accounting_code" invisible="1" groups="account.group_account_manager"/>
    </sheet>
</form>
```

### Security Best Practices

**1. Never Trust Client-Side Hiding**
- Hidden fields via `attrs` are hidden in UI only
- Always validate in Python that users can access the data
- Use `check_access_rights()` in model methods

Example:
```python
def get_cost_price(self):
    """Return cost price only if user has access."""
    self.check_access_rights('read')
    
    if not self.env.user.has_group('base.group_erp_manager'):
        raise AccessError("You don't have permission to view cost price")
    
    return self.cost_price
```

**2. Validate Permissions in Business Logic**
```python
def write(self, values):
    """Prevent unauthorized updates to sensitive fields."""
    if 'cost_price' in values:
        if not self.env.user.has_group('base.group_erp_manager'):
            raise AccessError("Only managers can update cost price")
    
    return super().write(values)
```

**3. Use Record Rules for Data Filtering**
```xml
<!-- security/sh_model_security.xml -->
<record id="sh_model_cost_rule" model="ir.rule">
    <field name="name">View Cost Only for Managers</field>
    <field name="model_id" ref="model_sh_custom_model"/>
    <field name="domain_force">[('state', '!=', 'draft')]</field>
    <field name="groups" eval="[(4, ref('base.group_erp_manager'))]"/>
</record>
```

### Common Patterns

**Pattern 1: Sensitive Numeric Fields**
```xml
<field name="ssn" groups="hr.group_hr_manager"/>
<field name="bank_account" groups="account.group_account_manager"/>
<field name="salary" groups="hr.group_hr_manager,hr.group_payroll_user"/>
```

**Pattern 2: Approval Fields (Visible only at certain states)**
```xml
<field name="approved_by" 
        attrs="{'invisible': [('state', 'not in', ['submitted', 'approved'])]}"/>
<field name="approval_date"
        attrs="{'readonly': [('state', 'in', ['approved', 'rejected'])],
                'invisible': [('approved', '=', False)]}"/>
```

**Pattern 3: Computed Access Levels**
```xml
<field name="sensitive_data" 
        attrs="{'invisible': [('user_access_level', '<', 5)]}"/>
<!-- Where user_access_level is computed based on groups -->
```

---

## 21. Data Migration & Field Evolution (Critical for Existing Data)

### Adding Required Fields to Models with Existing Data

**Problem**: When adding a new required field to a model that already has records, the migration will fail unless handled properly.

**Solution: 3-Step Approach**

**Step 1: Add field as optional in __init__.py upgrade**
```python
class ShInvoice(models.Model):
    _name = 'sh.invoice'
    
    # NEW FIELD - temporarily not required
    invoice_type = fields.Selection([
        ('standard', 'Standard'),
        ('proforma', 'Proforma'),
    ], string='Invoice Type', required=False, default='standard')
```

**Step 2: Use _auto_init() to backfill existing records**
```python
def _auto_init(self):
    """Backfill data for existing records during module upgrade."""
    result = super()._auto_init()
    
    # Backfill invoice_type for existing records
    self.env.cr.execute("""
        UPDATE sh_invoice 
        SET invoice_type = 'standard' 
        WHERE invoice_type IS NULL
    """)
    
    return result
```

**Step 3: In next version, make field required**
```python
# After all records have been backfilled in previous version
invoice_type = fields.Selection([
    ('standard', 'Standard'),
    ('proforma', 'Proforma'),
], string='Invoice Type', required=True, default='standard')
```

### Adding Related/Many2one Fields

**Pattern for required relationship to another model:**
```python
class ShInvoice(models.Model):
    _name = 'sh.invoice'
    
    # Method 1: Optional first, then required (safest)
    partner_category_id = fields.Many2one(
        'res.partner.category',
        string='Partner Category',
        required=False  # Phase 1
    )

# Upgrade step 1: Backfill with default
def _auto_init(self):
    result = super()._auto_init()
    
    # Find or create default category
    default_category = self.env['res.partner.category'].search(
        [('name', '=', 'General')], limit=1
    )
    if not default_category:
        default_category = self.env['res.partner.category'].create({
            'name': 'General'
        })
    
    # Backfill all records
    self.env.cr.execute("""
        UPDATE sh_invoice 
        SET partner_category_id = %s 
        WHERE partner_category_id IS NULL
    """, (default_category.id,))
    
    return result
```

### Adding Computed Fields with Store

**Safe pattern for computed fields that need history:**
```python
class ShInvoice(models.Model):
    _name = 'sh.invoice'
    
    amount = fields.Float()
    tax_rate = fields.Float()
    
    # Phase 1: Add as computed, NOT stored
    total_amount = fields.Float(
        compute='_compute_total_amount',
        store=False  # Don't store initially
    )
    
    @api.depends('amount', 'tax_rate')
    def _compute_total_amount(self):
        for record in self:
            record.total_amount = record.amount * (1 + record.tax_rate / 100)

# Phase 2 (next release): After verification, add store=True
total_amount = fields.Float(
    compute='_compute_total_amount',
    store=True  # Now store computed value
)
```

### Testing Data Migrations

```python
class TestDataMigration(TransactionCase):
    def test_backfill_new_required_field(self):
        """Test that migration backfills existing records correctly."""
        # Create old-style record (before field existed)
        self.env.cr.execute("""
            INSERT INTO sh_invoice (name, amount)
            VALUES ('Test', 100)
        """)
        
        # Simulate module upgrade
        self.env['sh.invoice']._auto_init()
        
        # Verify backfill worked
        invoice = self.env['sh.invoice'].search([('name', '=', 'Test')])
        self.assertIsNotNone(invoice.invoice_type)
        self.assertEqual(invoice.invoice_type, 'standard')
```

---

## 22. Error Handling & Recovery (Critical for Production)

### Graceful Error Handling Pattern

**Business Logic Errors (User-Facing)**
```python
from odoo.exceptions import UserError, ValidationError

def action_confirm(self):
    """Confirm with proper error handling."""
    # Check preconditions
    if not self.lines:
        raise UserError(_("Cannot confirm: No line items found"))
    
    if any(line.quantity <= 0 for line in self.lines):
        raise ValidationError(_("All line items must have quantity > 0"))
    
    # Perform operation in try-catch for unexpected errors
    try:
        self._create_accounting_entries()
        self.state = 'confirmed'
    except Exception as e:
        # Log the error for debugging
        _logger.exception("Failed to confirm invoice %s: %s", self.id, str(e))
        # Show user-friendly message
        raise UserError(_("Cannot confirm: %s. Please contact support.") % str(e))
```

### Database Transaction Safety

```python
def bulk_process_records(self):
    """Process records with transaction safety."""
    batch_size = 100
    offset = 0
    
    while True:
        records = self.search([], limit=batch_size, offset=offset)
        if not records:
            break
        
        # Process each record with error isolation
        for record in records:
            try:
                record.process()
            except Exception as e:
                # Log but don't stop batch
                _logger.exception(
                    "Failed to process record %s: %s", 
                    record.id, str(e)
                )
                # Optionally flag record as failed
                record.processing_error = str(e)[:500]
                self.env.cr.rollback()  # Rollback this record only
                continue
        
        # Commit batch
        self.env.cr.commit()
        offset += batch_size
        
        # Safety: avoid timeout
        if offset > 10000:
            _logger.warning("Batch processing limit reached")
            break
```

### API Error Handling

```python
@http.route('/api/invoice', type='json', auth='user')
def create_invoice(self, **kwargs):
    """Create invoice with error handling."""
    try:
        # Validate input
        if not kwargs.get('partner_id'):
            return {
                'error': 'Invalid request',
                'details': 'partner_id is required'
            }
        
        # Create record
        invoice = self.env['sh.invoice'].create({
            'partner_id': kwargs['partner_id'],
            'amount': kwargs.get('amount', 0),
        })
        
        return {
            'success': True,
            'invoice_id': invoice.id,
            'message': 'Invoice created'
        }
        
    except ValidationError as e:
        return {
            'error': 'Validation failed',
            'details': str(e)
        }
    except AccessError as e:
        return {
            'error': 'Access denied',
            'details': 'You do not have permission'
        }
    except Exception as e:
        _logger.exception("Unexpected error in create_invoice")
        return {
            'error': 'Internal error',
            'details': 'An unexpected error occurred'
        }
```

---

## 23. Concurrent Edit & Conflict Resolution

### Optimistic Locking Pattern (Recommended)

```python
class ShInvoice(models.Model):
    _name = 'sh.invoice'
    
    # Standard fields
    amount = fields.Float()
    notes = fields.Text()
    
    # Concurrency control
    _locking_date = 'write_date'  # Use Odoo's built-in write_date
    
    def write(self, values):
        """Prevent overwriting concurrent edits."""
        # Check if record was modified since last read
        if 'write_date' in self.env.context:
            client_write_date = self.env.context['write_date']
            
            for record in self:
                if record.write_date != client_write_date:
                    raise UserError(_(
                        "This record was modified by another user. "
                        "Please refresh and try again."
                    ))
        
        return super().write(values)
```

**Frontend: Send write_date with update request**
```javascript
// When fetching record
const invoice = await fetch('/api/invoice/123').then(r => r.json());
// Store write_date
const writeDate = invoice.write_date;

// When updating
fetch('/api/invoice/123', {
    method: 'POST',
    body: JSON.stringify({
        amount: 100,
        write_date: writeDate  // Send original write_date
    })
});
```

### Pessimistic Locking Pattern (For Critical Operations)

```python
def action_lock_for_approval(self):
    """Lock record for approval (prevents concurrent edits)."""
    self.env.cr.execute(
        "SELECT 1 FROM sh_invoice WHERE id = %s FOR UPDATE NOWAIT",
        [self.id]
    )
    # If we reach here, lock acquired successfully
    # Perform critical operation
    self.state = 'waiting_approval'
```

---

**Note**: These standards are **mandatory**. Code not following these guidelines will be rejected during code review.

For questions or clarifications, contact the senior development team.

---

**Changelog**:
- **v2.2 (2026-08-12) - Critical Fixes & High-Impact Additions**:
  - ✅ **FIXED: Section 1** — Clarified naming conventions: `state`, `create_date`, etc. are core Odoo fields, not to be avoided
  - ✅ **FIXED: Section 3** — Added multi-company + shared records pattern (company_id = False handling)
  - ✅ **FIXED: Section 19** — Corrected PDF template syntax error (use model method instead of inline Python slicing)
  - ✅ **FIXED: Section 20** — Added computed field security validation (server-side checks in compute methods)
  - ✅ **ENHANCED: Section 7** — Added complete archive action methods with validation and notifications
  - ✅ **NEW: Section 21** — Data Migration & Field Evolution (3-step approach for required fields, safe field addition)
  - ✅ **NEW: Section 22** — Error Handling & Recovery (business logic errors, transactions, API safety)
  - ✅ **NEW: Section 23** — Concurrent Edit & Conflict Resolution (optimistic & pessimistic locking)
  - ✅ **All critical issues resolved** — 4/4 critical issues fixed, 3 high-impact sections added

- **v2.1 (2026-08-12)**: 
  - Removed duplicate "Archive vs. Delete" section
  - Added **Section 19: PDF Report Design & Dynamic Values** — comprehensive guide for handling variable amounts, long text, line items with proper formatting and CSS
  - Added **Section 20: Field Visibility & Access-Based Controls** — automatic field hiding based on user groups defined in `ir.model.access.csv`, with practical examples and security best practices
  - All sections now interconnected for complete developer workflow
  
- **v2.0 (2026-08-12)**: Restructured for clarity; added comprehensive archive flow; fixed duplicate sections; improved technical accuracy; added validation checklist; corrected file header syntax; improved examples and code samples.

- **v1.0 (2026-02-14)**: Initial version