# H1 advertising inventory — implementation record

Project: **PROJ-0075 M4B Newspaper**  
Assignee: **Dhrumil**  
Prepared: **16 October 2026**  
Scope: committed H1.1–H1.4 changes through `e8e4a4d`.  
Scheduled H1 demo: **22 October 2026**.

H1 connects banner targeting to actual portal placements, restricts slot/size choices, and adds a sales inventory view. The four implementation tickets are committed. Deployment, migration application and browser/live-domain acceptance are not verified; this document does not mark the milestone accepted. Live banner delivery remains H2.

## 1. Commits and branches

| Ticket | Commit | Feature branch | Deliverable |
|---|---|---|---|
| H1.1 | `93dc2a9` | `feature/h1-1-slot-catalog` | Portal slot catalog and source coverage tests |
| H1.2 | `1d62de0` | `feature/h1-2-banner-slot-validation` | Banner slot field, migration and targeting/size validation |
| H1.3 | `95f900f` | `feature/h1-3-banner-slot-picker` | Authenticated catalog endpoint and constrained banner form |
| H1.4 | `e8e4a4d` | `feature/h1-4-ad-inventory` | Free/booked inventory API and dashboard page |

These branches follow the preceding feature branch in sequence, starting from H0.5. Push, MR and deployed branch state were not verified. Exact committed patches appear in section 9.

## 2. Changed files

| File | Milestone | Specific changed part |
|---|---|---|
| `backend/ads/api.py` | H1.2, H1.3, H1.4 | Validates banner targeting before atomic saves; adds authenticated GET /slots and GET /inventory. |
| `backend/ads/inventory.py` | H1.4 | Computes read-only free/booked rows using active campaign/banner date overlaps and portal targeting. |
| `backend/ads/migrations/0004_adbanner_slot_id.py` | H1.2 | Adds the slot_id column/index, defaulting existing banners to an empty string. |
| `backend/ads/models.py` | H1.2 | Added indexed AdBanner.slot_id; retained position for compatibility. |
| `backend/ads/schemas.py` | H1.2, H1.3, H1.4 | Requires slot_id on create, supports partial slot updates, returns slot_id; adds catalog and inventory response schemas. |
| `backend/ads/slot_validation.py` | H1.2 | Normalizes domain aliases and validates enabled slot/size availability across every supplied domain. |
| `backend/ads/slots.py` | H1.1 | New static SLOT_CATALOG, typed Slot/SlotSize records, three canonical domains, shared and portal-specific slot definitions. |
| `backend/ads/test_inventory.py` | H1.4 | Fourteen authenticated inventory tests cover overlap, targeting, UTC boundaries, validation and query counts. |
| `backend/ads/test_slots.py` | H1.1 | Five source contract tests compare enabled catalog entries with uncommented portal JSX, sizes and pages. |
| `backend/ads/tests.py` | H1.2, H1.3 | Adds nine targeting tests and a catalog contract test; updates fixtures and anonymous-read coverage. |
| `backend/core/tests.py` | H1.2 | Fixes a required organization field on the Site fixture used by advertiser deletion tests. |
| `frontend/package.json` | H1.3 | Adds test:ad-slots script; no dependency additions. |
| `frontend/scripts/test-ad-slots.cjs` | H1.3 | Runs Node assertions against the TypeScript shared-slot helper without launching the application. |
| `frontend/src/app/dashboard/ads/inventory/page.tsx` | H1.4 | New inventory page with dates, portal/status filters, sizes, booking details, edit links and refresh. |
| `frontend/src/components/ads/banner-form.tsx` | H1.3 | Replaces generic position/free dimensions with portals, shared enabled slots and allowed sizes; preserves valid edit selections. |
| `frontend/src/components/app-sidebar.tsx` | H1.4 | Adds the Inventario entry under advertising navigation. |
| `frontend/src/components/ui/multi-select.tsx` | H1.3 | Makes remove buttons non-submit buttons and adds accessible remove labels. |
| `frontend/src/lib/ad-slots.ts` | H1.3 | Adds catalog types, domain normalization, size keys and sharedSlots intersection logic. |

**18 unique files** changed across H1. No portal template, Compose or production configuration change is included.

## 3. H1.1 — real slot catalog

### `backend/ads/slots.py`

`SLOT_CATALOG` contains **38 portal-specific entries: 27 enabled and 11 disabled**. The source of truth is portal JSX, including commented placements. Each record has:

```python
class SlotSize(TypedDict):
    width: int
    height: int
    label: str

class Slot(TypedDict):
    site_domain: str
    slot_id: str
    label: str
    page: Literal["home", "article", "global"]
    desktop_sizes: list[SlotSize]
    mobile_sizes: list[SlotSize]
    enabled: bool
```

The canonical domains match Compose's non-www `PORTAL_DOMAIN` values:

- `laclavenoticias.es`
- `elobjetivonoticias.es`
- `eltitulardemurcia.es`

| Portal | Enabled | Disabled | Total |
|---|---:|---:|---:|
| La Clave | 7 | 5 | 12 |
| El Objetivo | 13 | 1 | 14 |
| El Titular | 7 | 5 | 12 |

The six common enabled placements are `home-top-billboard`, `home-after-hero`, `home-after-empresa`, `home-after-deportes`, `home-after-widgets-grid` and `home-sidebar-top`.

La Clave and El Titular also enable `home-before-footer`. Their middle/bottom sidebar, floating-left/right and hero-sidebar placements remain disabled.

El Objetivo also enables the widget-square, middle-halfpage and bottom-sticky sidebar slots, floating-left/right, `article-after-content`, and `global-footer-leaderboard`. Its hero-sidebar placement remains disabled. El Objetivo's right floating banner is **120×600**, while its left floating banner is **160×600**. Floating banners have no mobile size because their component is desktop-only.

Example generated entry:

```python
{
    "site_domain": "laclavenoticias.es",
    "slot_id": "home-sidebar-top",
    "label": "Portada · lateral superior",
    "page": "home",
    "desktop_sizes": [{"width": 300, "height": 250, "label": "300×250"}],
    "mobile_sizes": [{"width": 320, "height": 50, "label": "320×50"}],
    "enabled": True,
}
```

### `backend/ads/test_slots.py`

Five standard-library tests compare the enabled catalog with active `Banner` and `FloatingBanner` literal placements in each portal's:

- `src/pages/Index.tsx`
- `src/pages/ArticleDetail.tsx`
- `src/components/layout/Footer.tsx`
- `src/components/news/HeroSection.tsx`

The scanner ignores comments while preserving strings such as URLs. Tests check exact enabled placement sets, matching pages and desktop/mobile sizes, uniqueness and valid record fields. Synthetic cases prove that adding an unregistered placement or removing an enabled placement causes failure. Missing source files fail explicitly rather than silently skipping coverage.

The scanner covers the current literal JSX format in those templates; it is not a full TypeScript parser. The full checkout, including `portals/`, is required. It can run without Django/database access and is also discovered by `manage.py test ads`.

## 4. H1.2 — explicit banner slots and API validation

### Model and migration

`backend/ads/models.py` adds:

```python
slot_id = models.CharField(max_length=100, blank=True, default="", db_index=True, verbose_name=_("Hueco publicitario"))
```

`backend/ads/migrations/0004_adbanner_slot_id.py` adds this field after migration `0003_alter_adbanner_options_alter_adcampaign_options_and_more`. Existing rows receive `slot_id=""`; no slot is guessed from the old generic position. The `position`, `width` and `height` fields remain.

The migration was checked and exercised in isolated SQLite, not applied to production during this work. Old banners remain readable, but an API update must supply a valid slot and targeting before it can save.

### API schemas: `backend/ads/schemas.py`

```python
# AdBannerBase: required for creates
slot_id: str = Field(min_length=1, max_length=100)

# AdBannerUpdate: may be omitted in a partial update
slot_id: Optional[str] = Field(default=None, min_length=1, max_length=100)

# AdBannerOut: returned on create/read/update/list
slot_id: str
```

Explicit null slot values are rejected by resulting-state validation. Omitting the field during updates retains the stored slot.

### Catalog validation: `backend/ads/slot_validation.py`

```python
def normalize_site_domain(domain):
    return domain.strip().lower().rstrip(".").removeprefix("www.")

def validate_slot_targeting(slot_id, width, height, domains):
    if not slot_id or not isinstance(slot_id, str):
        raise ValueError("Selecciona un hueco publicitario válido.")
    if width is None or height is None or width <= 0 or height <= 0:
        raise ValueError("El ancho y el alto deben ser mayores que cero.")
    if not domains:
        raise ValueError("Selecciona al menos un portal o marca el banner como global.")

    for domain in domains:
        normalized = normalize_site_domain(domain)
        slot = next((entry for entry in SLOT_CATALOG
                     if entry["site_domain"] == normalized and entry["slot_id"] == slot_id
                     and entry["enabled"]), None)
        if slot is None:
            raise ValueError(
                f"El hueco {slot_id} no está habilitado en {normalized}. "
                "Selecciona un hueco común o portales explícitos."
            )
        allowed = {(size["width"], size["height"])
                   for size in slot["desktop_sizes"] + slot["mobile_sizes"]}
        if (width, height) not in allowed:
            sizes = ", ".join(f"{w}×{h}" for w, h in sorted(allowed))
            raise ValueError(
                f"El tamaño {width}×{height} no es válido para {slot_id} en {normalized}. "
                f"Tamaños permitidos: {sizes}."
            )
```

Every target domain must contain an enabled catalog entry for the slot, and the width/height pair must occur in its desktop or mobile size lists. The same pair is checked on every target. Tablet sizes are not separately accepted by this catalog contract.

For example, a 970×250 banner cannot use `home-sidebar-top`. `article-after-content` can target El Objetivo, but cannot target La Clave or be global. Domain comparison accepts case differences, surrounding whitespace, a trailing dot and the `www.` alias.

### Validated writes: `backend/ads/api.py`

`validate_banner_targets()` resolves selected Site IDs, rejects unknown IDs, checks targeting, and converts catalog errors into HTTP 422. Non-global banners require at least one valid selected portal. Global banners are checked against `PORTAL_DOMAINS`, regardless of the selected-site list, and store an empty explicit target list.

The update endpoint validates the resulting banner before saving:

```python
target_site_ids = data.pop("target_site_ids", list(banner.target_sites.values_list("id", flat=True)))
sites = validate_banner_targets(
    data.get("slot_id", banner.slot_id), data.get("width", banner.width),
    data.get("height", banner.height), data.get("is_global", banner.is_global), target_site_ids,
)
```

Create and update run inside `transaction.atomic()`; update locks the banner row. A failed validation does not change its name, dimensions, slot or many-to-many portal targeting. Global saves clear explicit targets. Existing H0 image/script validation still runs.

### Tests and fixture corrections

`backend/ads/tests.py` adds nine targeting tests: required/unknown/disabled slots, allowed-size round trips, every-target validation, global intersection, invalid targets, rejected partial updates, target/global changes, www aliases, and explicit slot assignment for legacy banners.

Fixtures now use a real catalog portal and valid slot. Clearing explicit targeting in the CRUD test also switches the banner to global mode, since an empty non-global target list is invalid.

The isolated run exposed a pre-existing fixture error: `Site.organization` is required. Site fixtures in both ads tests and `backend/core/tests.py` now include `organization=self.org`. This only changes test data.

## 5. H1.3 — catalog API and constrained banner form

### Catalog endpoint and response schemas

`backend/ads/api.py` adds this authenticated read-only endpoint:

```python
def list_slots(request):
    return SLOT_CATALOG
```

Its decorator is `@router.get("/slots", response=List[AdSlotOut])`, exposing **GET `/api/v1/ads/slots`** through the existing session-authenticated API. It returns the entire catalog, including disabled records. The form filters disabled entries out. The handler itself makes no database queries; session authentication can still query the database.

`AdSlotSizeOut` describes `{width, height, label}`. `AdSlotOut` describes the domain, slot identifier, Spanish label, page, desktop/mobile sizes and enabled flag.

### Shared selection logic: `frontend/src/lib/ad-slots.ts`

```tsx
export interface SlotSize {
    width: number
    height: number
    label: string
}

export interface AdSlot {
    site_domain: string
    slot_id: string
    label: string
    page: "home" | "article" | "global"
    desktop_sizes: SlotSize[]
    mobile_sizes: SlotSize[]
    enabled: boolean
}

export const normalizeSiteDomain = (domain: string) =>
    domain.trim().toLowerCase().replace(/\.$/, "").replace(/^www\./, "")

export const sizeKey = (size: Pick<SlotSize, "width" | "height">) => `${size.width}x${size.height}`

// A dimension pair may be desktop on one portal and mobile on another,
// matching the API's combined desktop/mobile size validation.
export function sharedSlots(catalog: AdSlot[], domains: string[]): (AdSlot & { sizes: SlotSize[] })[] {
    const targets = [...new Set(domains.map(normalizeSiteDomain))]
    if (!targets.length) return []
    return catalog.filter(slot => slot.enabled && slot.site_domain === targets[0]).flatMap(slot => {
        const entries = targets.map(domain => catalog.find(entry =>
            entry.enabled && entry.site_domain === domain && entry.slot_id === slot.slot_id))
        if (entries.some(entry => !entry)) return []
        const sizes = [...new Map([...slot.desktop_sizes, ...slot.mobile_sizes]
            .map(size => [sizeKey(size), size])).values()].filter(size => entries.every(entry =>
            [...entry!.desktop_sizes, ...entry!.mobile_sizes].some(other => sizeKey(other) === sizeKey(size))))
        return sizes.length ? [{ ...slot, sizes }] : []
    })
}
```

`sharedSlots()` normalizes selected hostnames, removes duplicate domains, excludes disabled/exclusive slots, and intersects dimension pairs. A slot with no common size is excluded. Empty or unknown selections produce no available slots. Combined desktop/mobile size validation matches the backend's behavior.

### Form: `frontend/src/components/ads/banner-form.tsx`

The form fetches `/api/v1/ads/slots`, includes a required `slot_id` in its schema/payload, and restores the stored slot on edit. It replaces the generic position control and freely editable dimensions with this flow:

1. Select explicit portals or global mode.
2. Select an enabled slot shared by those domains.
3. Select one of that slot's common allowed sizes.

For global mode, domains are derived from the full catalog, so the intersection includes all three portals. The selected size sets all three saved fields:

```tsx
form.setValue("width", size.width, { shouldDirty: true, shouldValidate: true });
form.setValue("height", size.height, { shouldDirty: true, shouldValidate: true });
form.setValue("size_label", size.label, { shouldDirty: true });
```

Changing portals invalidates exclusive slots. An effect clears an invalid `slot_id`, resets invalid width/height to zero and clears `size_label`. Valid existing selections are preserved. Save is blocked unless targeting, slot and size are valid. Loading, catalog error and no-common-slot messages explain unavailable choices. Saving still uses H0's CSRF-aware `apiFetch`.

Example: El Objetivo alone offers `article-after-content`; adding La Clave removes it. A salesperson cannot manually enter 970×250 for a 300×250 sidebar slot. Mobile catalog sizes may also be selected; this does not itself implement responsive public delivery.

The committed form contains labels describing locations. A schematic location miniature from the broader plan is not present in this commit.

### Portal selection and frontend assertions

`frontend/src/components/ui/multi-select.tsx` adds `type="button"` to remove buttons to prevent accidental form submission, plus an accessible label such as `Quitar La Clave`.

`frontend/package.json` adds:

```json
"test:ad-slots": "node scripts/test-ad-slots.cjs"
```

`frontend/scripts/test-ad-slots.cjs` transpiles only the pure TypeScript helper and asserts empty/unknown selections, alias normalization, portal changes, disabled/exclusive slots, incompatible dimensions, deduplication and three-portal global intersection. No dependency or lockfile change was needed.

The backend catalog contract checks the endpoint response against `SLOT_CATALOG`, includes disabled entries, and extends anonymous-read rejection to `/ads/slots`.

## 6. H1.4 — free/booked inventory

### Read-only inventory calculation: `backend/ads/inventory.py`

```python
def inventory_for_range(start_date, end_date):
    start = datetime.combine(start_date, time.min, tzinfo=datetime_timezone.utc)
    stop = datetime.combine(end_date + timedelta(days=1), time.min, tzinfo=datetime_timezone.utc)
    banners = AdBanner.objects.filter(
        is_active=True, campaign__is_active=True,
        campaign__start_date__lt=stop, campaign__end_date__gte=start,
    ).exclude(slot_id="").select_related("campaign").prefetch_related("target_sites")
    by_slot = {}
    for banner in banners:
        domains = None if banner.is_global else {
            normalize_site_domain(site.domain) for site in banner.target_sites.all()
        }
        booking = {
            "banner_id": banner.id, "banner_name": banner.name,
            "campaign_id": banner.campaign_id, "campaign_name": banner.campaign.name,
            "start_date": banner.campaign.start_date, "end_date": banner.campaign.end_date,
            "width": banner.width, "height": banner.height,
        }
        by_slot.setdefault(banner.slot_id, []).append((domains, booking))

    rows = []
    for slot in SLOT_CATALOG:
        if not slot["enabled"]:
            continue
        bookings = [booking for domains, booking in by_slot.get(slot["slot_id"], [])
                    if domains is None or slot["site_domain"] in domains]
        rows.append({**slot, "status": "booked" if bookings else "free", "bookings": bookings})
    return rows
```

The inventory contains one row per enabled portal/slot. Dates are inclusive UTC calendar dates, implemented using a midnight start and exclusive midnight after the end date.

A slot is **booked** when at least one banner:

- Is active and belongs to an active campaign.
- Has that `slot_id`.
- Has campaign dates overlapping the requested range.
- Is global, or explicitly targets that portal (with domain alias normalization).

Otherwise the slot is **free**. A booking ending exactly at the range start counts as overlapping; one starting at midnight after the final day does not. Future bookings appear when querying their range. Multiple creatives are all listed; the view does not prevent overlapping bookings.

`select_related("campaign")` and `prefetch_related("target_sites")` avoid queries per row. The populated-inventory query-count test asserts two queries, excluding authentication.

Disabled slots are not offered. Legacy banners with empty `slot_id` are excluded until explicitly assigned; this is explained on the page. Inventory shows scheduling occupancy, not evidence that a banner is live on a portal.

### Endpoint: `backend/ads/api.py`

The decorator is `@router.get("/inventory", response=List[AdInventoryOut])`:

```python
def list_inventory(request, start_date: date = None, end_date: date = None):
    start_date = start_date or timezone.now().date()
    end_date = end_date or start_date
    if start_date > end_date:
        raise HttpError(422, "La fecha de inicio debe ser anterior o igual a la fecha de fin.")
    # The exclusive upper bound must be representable by datetime.
    if end_date == date.max:
        raise HttpError(422, "La fecha de fin está fuera del rango permitido.")
    return inventory_for_range(start_date, end_date)
```

Example request:

```text
GET /api/v1/ads/inventory?start_date=2026-10-22&end_date=2026-10-31
```

This endpoint requires the existing dashboard session. Without parameters it defaults to today in UTC; an omitted end date defaults to the resolved start date. Invalid date syntax, reversed dates and the unrepresentable exclusive upper bound at `9999-12-31` return HTTP 422.

### Response schemas: `backend/ads/schemas.py`

`AdInventoryOut` extends `AdSlotOut` with `status: Literal["free", "booked"]` and a bookings list. Each booking includes banner/campaign IDs and names, campaign start/end datetimes and creative width/height. IDs support dashboard edit links without further requests.

### Page and sidebar

`frontend/src/app/dashboard/ads/inventory/page.tsx` adds **`/dashboard/ads/inventory`** with:

- Inclusive start/end UTC date inputs and an apply button.
- Portal and Libre/Ocupado filters.
- Counts of free, booked and visible slots.
- A table showing portal, slot label/ID, desktop/mobile sizes and status.
- Overlapping campaign/banner names, actual creative dimensions and UTC dates.
- Links to `/dashboard/ads/campaigns/{id}` and `/dashboard/ads/banners/{id}`.
- Loading/error/retry states, manual refresh and a 60-second SWR refresh interval.

Changing dates only applies after a valid submission; the displayed queried range identifies the data currently shown. Filters operate on the returned rows. Green/amber status labels also include text.

`frontend/src/components/app-sidebar.tsx` adds:

```tsx
{
    title: "Inventario",
    url: "/dashboard/ads/inventory",
    icon: IconReport,
}
```

### Inventory tests: `backend/ads/test_inventory.py`

Fourteen authenticated contract tests cover anonymous rejection, an all-free enabled catalog, targeted booking details/IDs, global bookings on three portals, paused campaign/banner exclusion, non-overlap, inclusive day boundaries and future dates, multi-day partial overlaps, multiple creatives, www/multi-portal targeting, disabled/legacy slot exclusion, invalid dates, default UTC dates and bounded query counts.

## 7. Verification and deployment status

| Check | Recorded result |
|---|---|
| H1.1 source contracts | Five tests passed in a temporary source copy, without Django/database/network access. |
| H1.2 ads/core contracts | 39 tests passed using isolated in-memory SQLite and network-blocked execution. |
| H1.2 migration consistency | Passed; no missing ads migrations. |
| Latest H1.4 ads/core run, including H1.3 contracts | **54 tests passed** using a temporary source copy, isolated settings, in-memory SQLite, clean environment and blocked network access. |
| H1.4 migration consistency | Passed; no missing ads migrations. |
| H1.4 inventory page type check | Passed with a temporary no-emit TypeScript configuration. |
| H1.4 focused ESLint | No errors; seven existing sidebar warnings. |
| H1.3 standalone frontend assertion script | Present in the commit; execution was not verified in this chat. |
| Git whitespace checks during implementation | Passed for H1.1, H1.2 and H1.4. |
| Browser salesperson flow / live-domain H1 demo | Not performed or verified. |
| PostgreSQL-specific deployment behavior | Not verified by SQLite tests. |
| Production migration application / deployed code | Not performed or verified. |
| ERP ticket status / push / MR state | Not inspected or changed as part of this record. |

The latest 54 backend tests comprise 24 ads contracts, 14 inventory contracts, five source catalog contracts and 11 core advertiser contracts. The isolated runner mounted the real core and ads routers with session authentication; it did not exercise all unrelated production routes/services.

No production containers were restarted, rebuilt, stopped or changed. No tests ran against the production database. Do not run test commands with production settings or credentials.

Before deployment, review and apply migration `0004_adbanner_slot_id` in the approved deployment process. H1.2 requires `slot_id` on new writes, so deploy it together with the H1.3 form. Explicitly assign slots to any legacy banners rather than guessing from `position`. The migration has not been applied to production by this work.

## 8. Acceptance and scope boundaries

The implementation provides the catalog, validated targeting, constrained form and inventory page. Remaining H1 acceptance is to demonstrate that a salesperson can select La Clave + El Titular, see only common enabled slots/sizes, and see an overlapping demo campaign marked occupied in the correct portal and dates. The broader plan's schematic location miniature is also not implemented in the committed H1.3 form.

H1 does not implement public banner delivery, click redirects, impression/click measurement, reports, consent or operational deployment work. Those remain H2–H5. SaaS, Stripe, full multi-tenancy and Frappe integration remain deferred.

Existing unrelated working-tree changes—including core media optimization, requirements/Compose changes, account/article UI, and portal files—are excluded. The exact patches below come only from the four H1 commits. This document is local documentation; docs are ignored by the repository unless explicitly allowed.

## 9. Exact committed code patches

These patches are copied directly from Git, grouped by milestone. They include all specific code edits and preserve commit order; the sections above describe their final combined behavior.

### H1.1 — `93dc2a9`: feat(ads): add portal slot catalog and source contract tests

```diff
diff --git a/backend/ads/slots.py b/backend/ads/slots.py
new file mode 100644
index 0000000..e99ec20
--- /dev/null
+++ b/backend/ads/slots.py
@@ -0,0 +1,86 @@
+"""Static portal inventory; commented JSX remains disabled.
+
+Domains match PORTAL_DOMAIN in Compose. Floating slots have no mobile size.
+"""
+from typing import Literal, TypedDict
+
+
+class SlotSize(TypedDict):
+    width: int
+    height: int
+    label: str
+
+
+class Slot(TypedDict):
+    site_domain: str
+    slot_id: str
+    label: str
+    page: Literal["home", "article", "global"]
+    desktop_sizes: list[SlotSize]
+    mobile_sizes: list[SlotSize]
+    enabled: bool
+
+
+PORTAL_DOMAINS = (
+    "laclavenoticias.es",
+    "elobjetivonoticias.es",
+    "eltitulardemurcia.es",
+)
+
+
+def _size(dimensions: str) -> SlotSize:
+    width, height = map(int, dimensions.split("x"))
+    return {"width": width, "height": height, "label": f"{width}×{height}"}
+
+
+def _slot(
+    domain: str, slot_id: str, label: str, desktop: str,
+    mobile: str | None = "320x50",
+    page: Literal["home", "article", "global"] = "home",
+    enabled: bool = True,
+) -> Slot:
+    return {
+        "site_domain": domain,
+        "slot_id": slot_id,
+        "label": label,
+        "page": page,
+        "desktop_sizes": [_size(desktop)],
+        "mobile_sizes": [_size(mobile)] if mobile else [],
+        "enabled": enabled,
+    }
+
+
+def _common_slots(domain: str) -> list[Slot]:
+    return [
+        _slot(domain, "home-top-billboard", "Portada · superior", "970x250"),
+        _slot(domain, "home-after-hero", "Portada · tras la apertura", "728x90"),
+        _slot(domain, "home-after-empresa", "Portada · tras Empresa", "300x250"),
+        _slot(domain, "home-after-deportes", "Portada · tras Deportes", "336x280"),
+        _slot(domain, "home-after-widgets-grid", "Portada · tras los widgets", "728x90"),
+        _slot(domain, "home-sidebar-top", "Portada · lateral superior", "300x250"),
+        _slot(domain, "home-hero-sidebar-top", "Portada · lateral de la apertura", "160x600", enabled=False),
+    ]
+
+
+def _clave_titular_slots(domain: str) -> list[Slot]:
+    return [
+        _slot(domain, "home-before-footer", "Portada · antes del pie", "728x90"),
+        _slot(domain, "home-sidebar-middle", "Portada · lateral central", "300x250", enabled=False),
+        _slot(domain, "home-sidebar-bottom", "Portada · lateral inferior", "250x250", enabled=False),
+        _slot(domain, "home-floating-left", "Portada · flotante izquierdo", "160x600", mobile=None, enabled=False),
+        _slot(domain, "home-floating-right", "Portada · flotante derecho", "160x600", mobile=None, enabled=False),
+    ]
+
+
+SLOT_CATALOG: list[Slot] = [
+    *(slot for domain in PORTAL_DOMAINS for slot in _common_slots(domain)),
+    *(slot for domain in ("laclavenoticias.es", "eltitulardemurcia.es")
+      for slot in _clave_titular_slots(domain)),
+    _slot("elobjetivonoticias.es", "home-sidebar-widget-square", "Portada · lateral cuadrado", "250x250", "250x250"),
+    _slot("elobjetivonoticias.es", "home-sidebar-middle-halfpage", "Portada · lateral media página", "300x600", "300x250"),
+    _slot("elobjetivonoticias.es", "home-sidebar-bottom-sticky", "Portada · lateral inferior fijo", "300x1050", "300x250"),
+    _slot("elobjetivonoticias.es", "home-floating-left", "Portada · flotante izquierdo", "160x600", mobile=None),
+    _slot("elobjetivonoticias.es", "home-floating-right", "Portada · flotante derecho", "120x600", mobile=None),
+    _slot("elobjetivonoticias.es", "article-after-content", "Artículo · tras el contenido", "728x90", "300x250", page="article"),
+    _slot("elobjetivonoticias.es", "global-footer-leaderboard", "Todas las páginas · pie", "728x90", page="global"),
+]
diff --git a/backend/ads/test_slots.py b/backend/ads/test_slots.py
new file mode 100644
index 0000000..0fe1b11
--- /dev/null
+++ b/backend/ads/test_slots.py
@@ -0,0 +1,140 @@
+"""Slot source contracts; no Django settings or database required.
+
+Standalone: python3 -m unittest discover -s backend/ads -p test_slots.py -v
+Also discovered by manage.py test ads. Requires a full checkout with portals/.
+"""
+import re
+import unittest
+from pathlib import Path
+
+if __package__:
+    from .slots import PORTAL_DOMAINS, SLOT_CATALOG
+else:
+    from slots import PORTAL_DOMAINS, SLOT_CATALOG
+
+
+REPO_ROOT = Path(__file__).resolve().parents[2]
+PORTAL_DIRECTORIES = {
+    "laclavenoticias.es": "laclavenoticias",
+    "elobjetivonoticias.es": "elobjetivonoticias",
+    "eltitulardemurcia.es": "eltitulardemurcia",
+}
+TEMPLATES = {
+    "src/pages/Index.tsx": "home",
+    "src/pages/ArticleDetail.tsx": "article",
+    "src/components/layout/Footer.tsx": "global",
+    "src/components/news/HeroSection.tsx": "home",
+}
+# Match strings first to preserve URLs and comment-like text inside props.
+TOKENS = re.compile(
+    r""""(?:\\.|[^"\\])*"|'(?:\\.|[^'\\])*'|\x60(?:\\.|[^\x60\\])*\x60|/\*[\s\S]*?\*/|//[^\n]*"""
+)
+BANNER_TAG = re.compile(r"<(?:Banner|FloatingBanner)\b[\s\S]*?/>")
+PLACEMENT = re.compile(r"""(?<![\w-])placement\s*=\s*["']([^"']+)["']""")
+SIZE = re.compile(r"""(desktop|mobile)\s*:\s*["'](\d+x\d+)["']""")
+FLOATING_SIZE = re.compile(r"""\bsize\s*=\s*["'](\d+x\d+)["']""")
+
+
+def strip_comments(source):
+    return TOKENS.sub(
+        lambda match: "" if match.group().startswith(("/*", "//")) else match.group(),
+        source,
+    )
+
+
+def banner_placements(source):
+    """Read literal placements on the current Banner/FloatingBanner JSX."""
+    result = {}
+    for tag in BANNER_TAG.findall(strip_comments(source)):
+        match = PLACEMENT.search(tag)
+        if not match:
+            continue
+        sizes = dict(SIZE.findall(tag))
+        if tag.startswith("<FloatingBanner"):
+            floating = FLOATING_SIZE.search(tag)
+            if floating:
+                sizes["desktop"] = floating.group(1)
+        result[match.group(1)] = sizes
+    return result
+
+
+class SlotCatalogTests(unittest.TestCase):
+    def assert_coverage(self, domain, active, enabled):
+        self.assertEqual(
+            set(active), set(enabled),
+            f"{domain}: missing active slots={sorted(set(active) - set(enabled))}; "
+            f"enabled slots absent from templates={sorted(set(enabled) - set(active))}",
+        )
+
+    def test_enabled_catalog_matches_active_portal_placements(self):
+        for domain, directory in PORTAL_DIRECTORIES.items():
+            with self.subTest(domain=domain):
+                active = {}
+                for relative, page in TEMPLATES.items():
+                    path = REPO_ROOT / "portals" / directory / relative
+                    self.assertTrue(path.is_file(), f"Required portal source missing: {path}")
+                    for slot_id, sizes in banner_placements(path.read_text()).items():
+                        self.assertNotIn(slot_id, active, f"Duplicate placement: {domain}/{slot_id}")
+                        active[slot_id] = (page, sizes)
+                self.assertTrue(active, f"No active placements found for {domain}")
+                enabled = {
+                    slot["slot_id"]: slot for slot in SLOT_CATALOG
+                    if slot["site_domain"] == domain and slot["enabled"]
+                }
+                self.assert_coverage(domain, active, enabled)
+                for slot_id, (page, sizes) in active.items():
+                    slot = enabled[slot_id]
+                    self.assertEqual(slot["page"], page)
+                    for device in ("desktop", "mobile"):
+                        expected = sizes.get(device)
+                        actual = {
+                            f"{size['width']}x{size['height']}"
+                            for size in slot[f"{device}_sizes"]
+                        }
+                        self.assertEqual(actual, {expected} if expected else set(),
+                                         f"{domain}/{slot_id}: {device} sizes differ")
+
+    def test_catalog_records_are_valid_and_unique(self):
+        keys = set()
+        self.assertEqual(set(PORTAL_DOMAINS), set(PORTAL_DIRECTORIES))
+        for slot in SLOT_CATALOG:
+            key = (slot["site_domain"], slot["slot_id"])
+            with self.subTest(key=key):
+                self.assertNotIn(key, keys)
+                keys.add(key)
+                self.assertIn(slot["site_domain"], PORTAL_DOMAINS)
+                self.assertTrue(slot["slot_id"])
+                self.assertTrue(slot["label"])
+                self.assertIn(slot["page"], ("home", "article", "global"))
+                self.assertIsInstance(slot["enabled"], bool)
+                self.assertTrue(slot["desktop_sizes"])
+                for size in slot["desktop_sizes"] + slot["mobile_sizes"]:
+                    self.assertGreater(size["width"], 0)
+                    self.assertGreater(size["height"], 0)
+                    self.assertTrue(size["label"])
+
+    def test_comments_are_ignored_and_literal_props_are_preserved(self):
+        source = """
+        {/* <Banner placement="commented-jsx" /> */}
+        /* <Banner placement="commented-block" /> */
+        // <Banner placement="commented-line" />
+        <Banner
+            placement='active'
+            link="https://example.com/offer"
+            sizes={{desktop: "300x250", mobile: "320x50"}}
+        />
+        <FloatingBanner placement="floating" size="160x600" />
+        """
+        self.assertEqual(banner_placements(source), {
+            "active": {"desktop": "300x250", "mobile": "320x50"},
+            "floating": {"desktop": "160x600"},
+        })
+
+    def test_new_unregistered_placement_fails_coverage(self):
+        active = banner_placements('<Banner placement="new-unregistered-slot" />')
+        with self.assertRaisesRegex(AssertionError, "new-unregistered-slot"):
+            self.assert_coverage("example.com", active, {})
+
+    def test_enabled_slot_removed_from_template_fails_coverage(self):
+        with self.assertRaisesRegex(AssertionError, "removed-slot"):
+            self.assert_coverage("example.com", {}, {"removed-slot": {}})
```

### H1.2 — `1d62de0`: feat(ads): validate banner slots and sizes across target portals

```diff
diff --git a/backend/ads/api.py b/backend/ads/api.py
index 6f69201..7b0726e 100644
--- a/backend/ads/api.py
+++ b/backend/ads/api.py
@@ -2,6 +2,7 @@ from django.shortcuts import get_object_or_404
 from ninja import Router
 from ninja.errors import HttpError
 from django.utils import timezone
+from django.db import transaction
 from typing import List
 from uuid import UUID
 from core.models import Brand, MediaAsset
@@ -12,6 +13,9 @@ from .schemas import (
     AdBannerCreate, AdBannerUpdate, AdBannerOut
 )
 
+from .slots import PORTAL_DOMAINS
+from .slot_validation import validate_slot_targeting
+
 router = Router(tags=["Advertising"])
 
 
@@ -36,6 +40,21 @@ def validate_banner(active, image_id, script_code):
         if active and not image.file:
             raise HttpError(422, "Un banner activo requiere una imagen.")
 
+def validate_banner_targets(slot_id, width, height, is_global, target_site_ids):
+    if is_global is None or target_site_ids is None:
+        raise HttpError(422, "El estado global y la lista de portales no pueden ser nulos.")
+    unique_ids = set(target_site_ids)
+    sites = list(Site.objects.filter(id__in=unique_ids))
+    if len(sites) != len(unique_ids):
+        raise HttpError(422, "Uno o varios portales seleccionados no existen.")
+    domains = PORTAL_DOMAINS if is_global else [site.domain for site in sites]
+    try:
+        validate_slot_targeting(slot_id, width, height, domains)
+    except ValueError as error:
+        raise HttpError(422, str(error)) from error
+    return [] if is_global else sites
+
+
 # --- Campaign Endpoints ---
 
 @router.get("/campaigns", response=List[AdCampaignOut])
@@ -92,15 +111,18 @@ def list_banners(request, campaign_id: UUID = None):
     return qs
 
 @router.post("/banners", response=AdBannerOut)
+@transaction.atomic
 def create_banner(request, payload: AdBannerCreate):
     validate_banner(payload.is_active, payload.image_id, payload.script_code)
     campaign = get_object_or_404(AdCampaign, id=payload.campaign_id)
     
-    # Handle M2M separately
-    target_site_ids = payload.target_site_ids
+    sites = validate_banner_targets(
+        payload.slot_id, payload.width, payload.height, payload.is_global, payload.target_site_ids
+    )
     
     banner = AdBanner.objects.create(
         campaign=campaign,
+        slot_id=payload.slot_id,
         name=payload.name,
         image_id=payload.image_id,
         link=payload.link or "",
@@ -113,9 +135,7 @@ def create_banner(request, payload: AdBannerCreate):
         is_active=payload.is_active
     )
     
-    if target_site_ids:
-        sites = Site.objects.filter(id__in=target_site_ids)
-        banner.target_sites.set(sites)
+    banner.target_sites.set(sites)
         
     return banner
 
@@ -124,21 +144,24 @@ def get_banner(request, banner_id: UUID):
     return get_object_or_404(AdBanner, id=banner_id)
 
 @router.put("/banners/{banner_id}", response=AdBannerOut)
+@transaction.atomic
 def update_banner(request, banner_id: UUID, payload: AdBannerUpdate):
-    banner = get_object_or_404(AdBanner, id=banner_id)
+    banner = get_object_or_404(AdBanner.objects.select_for_update(), id=banner_id)
     
     data = payload.dict(exclude_unset=True)
     validate_banner(data.get("is_active", banner.is_active), data.get("image_id", banner.image_id), data.get("script_code", banner.script_code))
-    target_site_ids = data.pop('target_site_ids', None)
+    target_site_ids = data.pop("target_site_ids", list(banner.target_sites.values_list("id", flat=True)))
+    sites = validate_banner_targets(
+        data.get("slot_id", banner.slot_id), data.get("width", banner.width),
+        data.get("height", banner.height), data.get("is_global", banner.is_global), target_site_ids,
+    )
     
     for attr, value in data.items():
         setattr(banner, attr, value)
     
     banner.save()
     
-    if target_site_ids is not None:
-        sites = Site.objects.filter(id__in=target_site_ids)
-        banner.target_sites.set(sites)
+    banner.target_sites.set(sites)
         
     return banner
 
diff --git a/backend/ads/migrations/0004_adbanner_slot_id.py b/backend/ads/migrations/0004_adbanner_slot_id.py
new file mode 100644
index 0000000..ff02b07
--- /dev/null
+++ b/backend/ads/migrations/0004_adbanner_slot_id.py
@@ -0,0 +1,18 @@
+from django.db import migrations, models
+
+
+class Migration(migrations.Migration):
+    dependencies = [
+        ("ads", "0003_alter_adbanner_options_alter_adcampaign_options_and_more"),
+    ]
+
+    operations = [
+        migrations.AddField(
+            model_name="adbanner",
+            name="slot_id",
+            field=models.CharField(
+                max_length=100, blank=True, default="", db_index=True,
+                verbose_name="Hueco publicitario",
+            ),
+        ),
+    ]
diff --git a/backend/ads/models.py b/backend/ads/models.py
index 6e7e50b..68751d7 100644
--- a/backend/ads/models.py
+++ b/backend/ads/models.py
@@ -54,6 +54,7 @@ class AdBanner(TimeStampedModel):
     position = models.CharField(max_length=20, choices=POSITION_CHOICES, default='sidebar', verbose_name=_("Posición"))
     
     # Targeting
+    slot_id = models.CharField(max_length=100, blank=True, default="", db_index=True, verbose_name=_("Hueco publicitario"))
     is_global = models.BooleanField(default=False, verbose_name=_("Global (Todos los portales)"))
     target_sites = models.ManyToManyField(Site, blank=True, related_name='ads', verbose_name=_("Portales Destino (Si no es global)"))
     
diff --git a/backend/ads/schemas.py b/backend/ads/schemas.py
index d48e63b..7b2196e 100644
--- a/backend/ads/schemas.py
+++ b/backend/ads/schemas.py
@@ -1,4 +1,5 @@
 from ninja import Schema, ModelSchema
+from pydantic import Field
 from typing import List, Optional
 from datetime import datetime
 from uuid import UUID
@@ -38,6 +39,7 @@ class AdCampaignOut(AdCampaignBase):
 # --- AdBanner Schemas ---
 
 class AdBannerBase(Schema):
+    slot_id: str = Field(min_length=1, max_length=100)
     campaign_id: UUID
     name: str = "Banner"
     image_id: Optional[UUID] = None
@@ -55,6 +57,7 @@ class AdBannerCreate(AdBannerBase):
     pass
 
 class AdBannerUpdate(Schema):
+    slot_id: Optional[str] = Field(default=None, min_length=1, max_length=100)
     campaign_id: Optional[UUID] = None
     name: Optional[str] = None
     image_id: Optional[UUID] = None
@@ -69,6 +72,7 @@ class AdBannerUpdate(Schema):
     is_active: Optional[bool] = None
 
 class AdBannerOut(Schema):
+    slot_id: str
     id: UUID
     campaign_id: UUID
     campaign_name: str = None
diff --git a/backend/ads/slot_validation.py b/backend/ads/slot_validation.py
new file mode 100644
index 0000000..d04495d
--- /dev/null
+++ b/backend/ads/slot_validation.py
@@ -0,0 +1,34 @@
+"""Catalog validation independent of Django and database settings."""
+from .slots import SLOT_CATALOG
+
+
+def normalize_site_domain(domain):
+    return domain.strip().lower().rstrip(".").removeprefix("www.")
+
+
+def validate_slot_targeting(slot_id, width, height, domains):
+    if not slot_id or not isinstance(slot_id, str):
+        raise ValueError("Selecciona un hueco publicitario válido.")
+    if width is None or height is None or width <= 0 or height <= 0:
+        raise ValueError("El ancho y el alto deben ser mayores que cero.")
+    if not domains:
+        raise ValueError("Selecciona al menos un portal o marca el banner como global.")
+
+    for domain in domains:
+        normalized = normalize_site_domain(domain)
+        slot = next((entry for entry in SLOT_CATALOG
+                     if entry["site_domain"] == normalized and entry["slot_id"] == slot_id
+                     and entry["enabled"]), None)
+        if slot is None:
+            raise ValueError(
+                f"El hueco {slot_id} no está habilitado en {normalized}. "
+                "Selecciona un hueco común o portales explícitos."
+            )
+        allowed = {(size["width"], size["height"])
+                   for size in slot["desktop_sizes"] + slot["mobile_sizes"]}
+        if (width, height) not in allowed:
+            sizes = ", ".join(f"{w}×{h}" for w, h in sorted(allowed))
+            raise ValueError(
+                f"El tamaño {width}×{height} no es válido para {slot_id} en {normalized}. "
+                f"Tamaños permitidos: {sizes}."
+            )
diff --git a/backend/ads/tests.py b/backend/ads/tests.py
index 44b62e3..865a41d 100644
--- a/backend/ads/tests.py
+++ b/backend/ads/tests.py
@@ -25,7 +25,7 @@ class AdsApiContractTests(TestCase):
         self.user = User.objects.create_user(username="sales", password="test-password")
         UserProfile.objects.create(user=self.user, organization=self.org)
         self.brand = Brand.objects.create(name="Advertiser", organization=self.org)
-        self.site = Site.objects.create(name="Portal", domain="portal.example.com")
+        self.site = Site.objects.create(organization=self.org, name="Portal", domain="laclavenoticias.es")
         self.start = timezone.now().replace(microsecond=0)
         self.end = self.start + timedelta(days=30)
         self.image = MediaAsset.objects.create(file="contract-banner.gif")
@@ -59,7 +59,7 @@ class AdsApiContractTests(TestCase):
         return {
             "campaign_id": str(campaign.id), "name": "Banner",
             "image_id": str(self.image.id), "width": 300, "height": 250,
-            "position": "sidebar", "link": "https://example.com/offer",
+            "slot_id": "home-sidebar-top", "position": "sidebar", "link": "https://example.com/offer",
             "target_site_ids": [str(self.site.id)], "is_active": True, **changes,
         }
 
@@ -102,7 +102,7 @@ class AdsApiContractTests(TestCase):
         response = self.client.get(f"/api/v1/ads/banners?campaign_id={campaign.id}")
         self.assertEqual(response.status_code, 200)
         self.assertEqual([row["id"] for row in response.json()], [data["id"]])
-        response = self.write("put", path, {"name": "Edited", "target_site_ids": []})
+        response = self.write("put", path, {"name": "Edited", "target_site_ids": [], "is_global": True})
         self.assertEqual(response.status_code, 200)
         response = self.client.get("/api/v1/" + path)
         self.assertEqual(response.status_code, 200)
@@ -199,7 +199,8 @@ class AdsApiContractTests(TestCase):
 
     def test_banner_updates_validate_resulting_state(self):
         campaign = self.campaign()
-        banner = AdBanner.objects.create(campaign=campaign, is_active=False)
+        banner = AdBanner.objects.create(campaign=campaign, is_active=False, slot_id="home-sidebar-top")
+        banner.target_sites.add(self.site)
         path = f"ads/banners/{banner.id}"
         self.assertEqual(self.write("put", path, {"is_active": True}).status_code, 422)
         banner.refresh_from_db()
@@ -272,3 +273,115 @@ class AdsApiContractTests(TestCase):
         self.assertEqual(response.status_code, 200)
         self.assertEqual(response.json()["name"], "Edited creative")
         self.assertEqual(response.json()["image_id"], image_id)
+
+    def test_slot_is_required_and_unknown_or_disabled_slots_rejected(self):
+        campaign = self.campaign()
+        data = self.banner_data(campaign)
+        del data["slot_id"]
+        self.assertEqual(self.write("post", "ads/banners", data).status_code, 422)
+        for slot_id in ["", "unknown-slot", "home-sidebar-middle", None]:
+            with self.subTest(slot_id=slot_id):
+                self.assertEqual(self.write("post", "ads/banners", self.banner_data(campaign, slot_id=slot_id)).status_code, 422)
+        self.assertFalse(AdBanner.objects.exists())
+
+    def test_slot_and_size_round_trip(self):
+        campaign = self.campaign()
+        for width, height in [(300, 250), (320, 50)]:
+            response = self.write("post", "ads/banners", self.banner_data(campaign, width=width, height=height))
+            self.assertEqual(response.status_code, 200)
+            self.assertEqual(response.json()["slot_id"], "home-sidebar-top")
+            banner = AdBanner.objects.get(id=response.json()["id"])
+            self.assertEqual(banner.slot_id, "home-sidebar-top")
+            response = self.client.get(f"/api/v1/ads/banners/{banner.id}")
+            self.assertEqual(response.status_code, 200)
+            self.assertEqual(response.json()["slot_id"], banner.slot_id)
+        response = self.write("post", "ads/banners", self.banner_data(campaign, width=970, height=250))
+        self.assertEqual(response.status_code, 422)
+        self.assertIn("Tamaños permitidos", response.json()["detail"])
+        self.assertEqual(AdBanner.objects.count(), 2)
+
+    def test_slot_must_be_enabled_on_every_target_portal(self):
+        campaign = self.campaign()
+        objective = Site.objects.create(organization=self.org, name="El Objetivo", domain="elobjetivonoticias.es")
+        data = self.banner_data(campaign, slot_id="article-after-content", width=728, height=90,
+                                target_site_ids=[str(objective.id)])
+        self.assertEqual(self.write("post", "ads/banners", data).status_code, 200)
+        data["target_site_ids"].append(str(self.site.id))
+        self.assertEqual(self.write("post", "ads/banners", data).status_code, 422)
+        self.assertEqual(AdBanner.objects.count(), 1)
+
+    def test_global_banner_requires_common_slot_and_size(self):
+        campaign = self.campaign()
+        response = self.write("post", "ads/banners", self.banner_data(campaign, is_global=True, target_site_ids=[]))
+        self.assertEqual(response.status_code, 200)
+        self.assertTrue(response.json()["is_global"])
+        self.assertEqual(response.json()["target_sites"], [])
+        for slot_id, width, height in [("article-after-content", 728, 90), ("home-before-footer", 728, 90), ("home-floating-right", 120, 600), ("home-sidebar-top", 970, 250)]:
+            with self.subTest(slot_id=slot_id):
+                data = self.banner_data(campaign, slot_id=slot_id, width=width, height=height, is_global=True, target_site_ids=[])
+                self.assertEqual(self.write("post", "ads/banners", data).status_code, 422)
+        self.assertEqual(AdBanner.objects.count(), 1)
+
+    def test_targets_must_exist_and_belong_to_catalog(self):
+        campaign = self.campaign()
+        unknown = Site.objects.create(organization=self.org, name="Unknown", domain="unknown.example.com")
+        for ids in [[], [str(uuid4())], [str(self.site.id), str(uuid4())], [str(unknown.id)]]:
+            with self.subTest(ids=ids):
+                data = self.banner_data(campaign, target_site_ids=ids)
+                self.assertEqual(self.write("post", "ads/banners", data).status_code, 422)
+        self.assertFalse(AdBanner.objects.exists())
+
+    def test_slot_partial_updates_preserve_data_on_rejection(self):
+        campaign = self.campaign()
+        response = self.write("post", "ads/banners", self.banner_data(campaign))
+        self.assertEqual(response.status_code, 200)
+        banner = AdBanner.objects.get(id=response.json()["id"])
+        path = f"ads/banners/{banner.id}"
+        for data in [{"width": 970}, {"height": 90}, {"slot_id": "article-after-content"},
+                     {"target_site_ids": []}, {"target_site_ids": [str(uuid4())]},
+                     {"slot_id": None}, {"width": None}, {"height": None},
+                     {"is_global": None}, {"target_site_ids": None}]:
+            with self.subTest(data=data):
+                self.assertEqual(self.write("put", path, {"name": "Changed", **data}).status_code, 422)
+                banner.refresh_from_db()
+                self.assertEqual(banner.name, "Banner")
+                self.assertEqual(banner.slot_id, "home-sidebar-top")
+                self.assertEqual((banner.width, banner.height), (300, 250))
+                self.assertEqual(list(banner.target_sites.all()), [self.site])
+        self.assertEqual(self.write("put", path, {"name": "Edited"}).status_code, 200)
+        self.assertEqual(self.write("put", path, {"slot_id": "home-after-hero", "width": 728, "height": 90}).status_code, 200)
+        banner.refresh_from_db()
+        self.assertEqual(banner.slot_id, "home-after-hero")
+
+    def test_target_and_global_updates_validate_existing_slot(self):
+        campaign = self.campaign()
+        objective = Site.objects.create(organization=self.org, name="El Objetivo", domain="elobjetivonoticias.es")
+        response = self.write("post", "ads/banners", self.banner_data(campaign,
+            slot_id="article-after-content", width=728, height=90, target_site_ids=[str(objective.id)]))
+        self.assertEqual(response.status_code, 200)
+        banner = AdBanner.objects.get(id=response.json()["id"])
+        path = f"ads/banners/{banner.id}"
+        for data in [{"is_global": True}, {"target_site_ids": [str(objective.id), str(self.site.id)]}]:
+            self.assertEqual(self.write("put", path, data).status_code, 422)
+            banner.refresh_from_db()
+            self.assertFalse(banner.is_global)
+            self.assertEqual(list(banner.target_sites.all()), [objective])
+        response = self.write("put", path, {"slot_id": "home-after-hero", "is_global": True})
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.json()["target_sites"], [])
+        self.assertEqual(self.write("put", path, {"is_global": False}).status_code, 422)
+
+    def test_www_domain_alias_is_supported(self):
+        self.site.domain = "www.laclavenoticias.es"
+        self.site.save()
+        response = self.write("post", "ads/banners", self.banner_data(self.campaign()))
+        self.assertEqual(response.status_code, 200)
+
+    def test_legacy_banner_needs_explicit_slot_on_update(self):
+        campaign = self.campaign()
+        banner = AdBanner.objects.create(campaign=campaign, image=self.image)
+        banner.target_sites.add(self.site)
+        path = f"ads/banners/{banner.id}"
+        self.assertEqual(self.client.get("/api/v1/" + path).json()["slot_id"], "")
+        self.assertEqual(self.write("put", path, {"name": "Changed"}).status_code, 422)
+        self.assertEqual(self.write("put", path, {"slot_id": "home-sidebar-top"}).status_code, 200)
diff --git a/backend/core/tests.py b/backend/core/tests.py
index c57e1cd..cc86c79 100644
--- a/backend/core/tests.py
+++ b/backend/core/tests.py
@@ -63,7 +63,7 @@ class AdvertiserApiTests(TestCase):
     def campaign_tree(self, active):
         campaign = AdCampaign.objects.create(brand=self.brand, name="Campaign", start_date=timezone.now(), end_date=timezone.now() + timedelta(days=30), is_active=active)
         banner = AdBanner.objects.create(campaign=campaign)
-        site = Site.objects.create(name="Portal", domain="portal.example.com")
+        site = Site.objects.create(organization=self.org, name="Portal", domain="portal.example.com")
         metric = AdMetric.objects.create(banner=banner, site=site, date=timezone.now().date(), impressions=7)
         return campaign, banner, metric
```

### H1.3 — `95f900f`: feat(ads): add slot catalog and shared portal slot picker

```diff
diff --git a/backend/ads/api.py b/backend/ads/api.py
index 7b0726e..627382b 100644
--- a/backend/ads/api.py
+++ b/backend/ads/api.py
@@ -10,10 +10,10 @@ from publishing.models import Site
 from .models import AdCampaign, AdBanner
 from .schemas import (
     AdCampaignCreate, AdCampaignUpdate, AdCampaignOut,
-    AdBannerCreate, AdBannerUpdate, AdBannerOut
+    AdBannerCreate, AdBannerUpdate, AdBannerOut, AdSlotOut
 )
 
-from .slots import PORTAL_DOMAINS
+from .slots import PORTAL_DOMAINS, SLOT_CATALOG
 from .slot_validation import validate_slot_targeting
 
 router = Router(tags=["Advertising"])
@@ -55,6 +55,11 @@ def validate_banner_targets(slot_id, width, height, is_global, target_site_ids):
     return [] if is_global else sites
 
 
+@router.get("/slots", response=List[AdSlotOut])
+def list_slots(request):
+    return SLOT_CATALOG
+
+
 # --- Campaign Endpoints ---
 
 @router.get("/campaigns", response=List[AdCampaignOut])
diff --git a/backend/ads/schemas.py b/backend/ads/schemas.py
index 7b2196e..d5acc12 100644
--- a/backend/ads/schemas.py
+++ b/backend/ads/schemas.py
@@ -1,6 +1,6 @@
 from ninja import Schema, ModelSchema
 from pydantic import Field
-from typing import List, Optional
+from typing import List, Optional, Literal
 from datetime import datetime
 from uuid import UUID
 from .models import AdCampaign, AdBanner
@@ -104,3 +104,19 @@ class AdBannerOut(Schema):
     @staticmethod
     def resolve_target_sites(obj):
         return [{"id": s.id, "domain": s.domain} for s in obj.target_sites.all()]
+
+
+class AdSlotSizeOut(Schema):
+    width: int
+    height: int
+    label: str
+
+
+class AdSlotOut(Schema):
+    site_domain: str
+    slot_id: str
+    label: str
+    page: Literal["home", "article", "global"]
+    desktop_sizes: List[AdSlotSizeOut]
+    mobile_sizes: List[AdSlotSizeOut]
+    enabled: bool
diff --git a/backend/ads/tests.py b/backend/ads/tests.py
index 865a41d..6cfc40a 100644
--- a/backend/ads/tests.py
+++ b/backend/ads/tests.py
@@ -63,6 +63,16 @@ class AdsApiContractTests(TestCase):
             "target_site_ids": [str(self.site.id)], "is_active": True, **changes,
         }
 
+    def test_slot_catalog_includes_enabled_and_disabled_entries_without_queries(self):
+        from ads.api import list_slots
+        from ads.slots import SLOT_CATALOG
+        with self.assertNumQueries(0):
+            self.assertEqual(list_slots(None), SLOT_CATALOG)
+        response = self.client.get("/api/v1/ads/slots")
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.json(), SLOT_CATALOG)
+        self.assertTrue(any(not slot["enabled"] for slot in response.json()))
+
     def test_campaign_crud(self):
         response = self.write("post", "ads/campaigns", self.campaign_data())
         self.assertEqual(response.status_code, 200)
@@ -117,7 +127,7 @@ class AdsApiContractTests(TestCase):
         campaign = self.campaign()
         banner = AdBanner.objects.create(campaign=campaign, is_active=False)
         anonymous = Client()
-        for path in ["ads/campaigns", f"ads/campaigns/{campaign.id}", "ads/banners", f"ads/banners/{banner.id}"]:
+        for path in ["ads/slots", "ads/campaigns", f"ads/campaigns/{campaign.id}", "ads/banners", f"ads/banners/{banner.id}"]:
             with self.subTest(path=path):
                 self.assertEqual(anonymous.get("/api/v1/" + path).status_code, 401)
         for method, path, data in self.mutations(campaign, banner):
diff --git a/frontend/package.json b/frontend/package.json
index a048358..74124d2 100644
--- a/frontend/package.json
+++ b/frontend/package.json
@@ -7,6 +7,7 @@
     "build": "next build",
     "start": "next start",
     "lint": "eslint",
+    "test:ad-slots": "node scripts/test-ad-slots.cjs",
     "generate:pitch-deck": "bash scripts/generate-pitch-deck-pdf.sh"
   },
   "dependencies": {
diff --git a/frontend/scripts/test-ad-slots.cjs b/frontend/scripts/test-ad-slots.cjs
new file mode 100644
index 0000000..ec777f5
--- /dev/null
+++ b/frontend/scripts/test-ad-slots.cjs
@@ -0,0 +1,39 @@
+const assert = require('node:assert/strict');
+const fs = require('node:fs');
+const ts = require('typescript');
+const Module = require('node:module');
+const filename = require('node:path').resolve(__dirname, '../src/lib/ad-slots.ts');
+const compiled = ts.transpileModule(fs.readFileSync(filename, 'utf8'), {
+    compilerOptions: { module: ts.ModuleKind.CommonJS, target: ts.ScriptTarget.ES2017 },
+}).outputText;
+const loaded = new Module(filename, module);
+loaded._compile(compiled, filename);
+const { sharedSlots, sizeKey } = loaded.exports;
+const size = (width, height) => ({ width, height, label: `${width}×${height}` });
+const slot = (domain, id, desktop, mobile = [], enabled = true) => ({
+    site_domain: domain, slot_id: id, label: id, page: 'home',
+    desktop_sizes: desktop, mobile_sizes: mobile, enabled,
+});
+const catalog = [
+    slot('a.es', 'common', [size(728, 90)], [size(320, 50)]),
+    slot('b.es', 'common', [size(970, 250)], [size(320, 50)]),
+    slot('c.es', 'common', [size(320, 50)]),
+    slot('a.es', 'exclusive', [size(300, 250)]),
+    slot('a.es', 'disabled', [size(300, 250)]),
+    slot('b.es', 'disabled', [size(300, 250)], [], false),
+    slot('a.es', 'incompatible', [size(300, 250)]),
+    slot('b.es', 'incompatible', [size(300, 600)]),
+];
+assert.deepEqual(sharedSlots(catalog, []), []);
+assert.deepEqual(sharedSlots(catalog, ['unknown.es']), []);
+assert.deepEqual(sharedSlots(catalog, ['a.es', 'unknown.es']), []);
+const shared = sharedSlots(catalog, [' WWW.A.ES. ', 'b.es', 'a.es']);
+assert.deepEqual(shared.map(s => s.slot_id), ['common']);
+assert.deepEqual(shared[0].sizes.map(sizeKey), ['320x50']);
+assert.equal(sharedSlots(catalog, ['a.es']).some(s => s.slot_id === 'exclusive'), true);
+assert.equal(shared.some(s => s.slot_id === 'exclusive'), false);
+assert.equal(shared[0].sizes.some(s => sizeKey(s) === '728x90'), false);
+assert.equal(shared[0].sizes.some(s => sizeKey(s) === '320x50'), true);
+const global = sharedSlots(catalog, [...new Set(catalog.map(s => s.site_domain))]);
+assert.deepEqual(global.map(s => [s.slot_id, s.sizes.map(sizeKey)]), [['common', ['320x50']]]);
+console.log('Slot intersection checks passed: portal changes, global mode, disabled slots, dimensions, aliases, empty and unknown targets.');
diff --git a/frontend/src/components/ads/banner-form.tsx b/frontend/src/components/ads/banner-form.tsx
index 2a0d5ad..1285b1d 100644
--- a/frontend/src/components/ads/banner-form.tsx
+++ b/frontend/src/components/ads/banner-form.tsx
@@ -1,6 +1,6 @@
 "use client"
 
-import { useEffect, useState } from "react"
+import { useEffect, useMemo, useState } from "react"
 import { useRouter } from "next/navigation"
 import { zodResolver } from "@hookform/resolvers/zod"
 import { useForm } from "react-hook-form"
@@ -33,8 +33,10 @@ import { MultiSelect } from "@/components/ui/multi-select"
 
 import { apiFetch } from "@/lib/api-fetch"
 import { fetcher } from "@/lib/fetcher"
+import { AdSlot, sharedSlots, sizeKey } from "@/lib/ad-slots"
 
 const formSchema = z.object({
+    slot_id: z.string().min(1, "Selecciona un hueco publicitario."),
     campaign_id: z.string().uuid("Selecciona una campaña."),
     name: z.string().min(2, "El nombre es requerido."),
     width: z.coerce.number().min(1, "Ancho requerido."),
@@ -47,6 +49,9 @@ const formSchema = z.object({
     is_global: z.boolean().default(false),
     target_site_ids: z.array(z.string()).optional(),
     is_active: z.boolean().default(true),
+}).refine(values => values.is_global || !!values.target_site_ids?.length, {
+    message: "Selecciona al menos un portal.",
+    path: ["target_site_ids"],
 }).refine(values => !values.is_active || !!values.image_id, {
     message: "Un banner activo requiere una imagen.",
     path: ["image_id"],
@@ -64,36 +69,27 @@ export function BannerForm({ initialData }: BannerFormProps) {
     const { data: campaigns } = useSWR<any[]>("/api/v1/ads/campaigns", fetcher)
     const { data: sites } = useSWR<any[]>("/api/v1/publishing/sites", fetcher)
 
+    const { data: slots, error: slotsError } = useSWR<AdSlot[]>("/api/v1/ads/slots", fetcher)
+
     const form = useForm<z.infer<typeof formSchema>>({
         resolver: zodResolver(formSchema),
         defaultValues: {
+            slot_id: initialData?.slot_id || "",
             campaign_id: initialData?.campaign_id || "",
             name: initialData?.name || "",
-            width: initialData?.width || 300,
-            height: initialData?.height || 250,
+            width: initialData?.width || 0,
+            height: initialData?.height || 0,
             size_label: initialData?.size_label || "",
             position: initialData?.position || "sidebar",
             link: initialData?.link || "",
             script_code: "",
             image_id: initialData?.image_id || null,
             is_global: initialData?.is_global !== undefined ? initialData.is_global : false,
-            target_site_ids: initialData?.target_sites ? initialData.target_sites.map((s: any) => s.id || s.domain) : [], // check format
-            // The API returns target_sites as list of objects {id, domain} usually?
-            // My schema says List of {id, domain}.
-            // So mapping needs to be careful. initialData might have full objects.
+            target_site_ids: initialData?.target_sites ? initialData.target_sites.map((s: any) => String(s.id)) : [],
             is_active: initialData?.is_active !== undefined ? initialData.is_active : true,
         },
     })
 
-    // Fix initialData mapping for target_site_ids if needed
-    useEffect(() => {
-        if (initialData?.target_sites && Array.isArray(initialData.target_sites)) {
-            const ids = initialData.target_sites.map((s: any) => s.id);
-            form.setValue("target_site_ids", ids);
-        }
-    }, [initialData, form]);
-
-
     // Image Upload Logic
     const [uploading, setUploading] = useState(false);
     const [previewImage, setPreviewImage] = useState<string | null>(initialData?.image_url || null);
@@ -130,7 +126,36 @@ export function BannerForm({ initialData }: BannerFormProps) {
 
     const isGlobal = form.watch("is_global");
 
+    const targetIds = form.watch("target_site_ids") || [];
+    const slotId = form.watch("slot_id");
+    const width = form.watch("width");
+    const height = form.watch("height");
+    const domains = isGlobal
+        ? [...new Set(slots?.map(slot => slot.site_domain) || [])]
+        : targetIds.map(id => sites?.find(site => String(site.id) === id)?.domain || "");
+    const domainKey = JSON.stringify(domains);
+    const availableSlots = useMemo(() => sharedSlots(slots || [], JSON.parse(domainKey)), [slots, domainKey]);
+    const selectedSlot = availableSlots.find(slot => slot.slot_id === slotId);
+    const selectedSize = selectedSlot?.sizes.find(size => size.width === width && size.height === height);
+    const targetingReady = !!slots && (isGlobal || !!sites);
+
+    useEffect(() => {
+        if (!targetingReady) return;
+        if (slotId && !selectedSlot) {
+            form.setValue("slot_id", "", { shouldValidate: true });
+        }
+        if (!selectedSize) {
+            if (width) form.setValue("width", 0);
+            if (height) form.setValue("height", 0);
+            form.setValue("size_label", "");
+        }
+    }, [targetingReady, slotId, selectedSlot, selectedSize, width, height, form]);
+
     async function onSubmit(values: z.infer<typeof formSchema>) {
+        if (!targetingReady || !selectedSlot || !selectedSize) {
+            form.setError("slot_id", { message: "Selecciona un hueco y tamaño válidos para los portales." });
+            return;
+        }
         setIsSubmitting(true)
         try {
             const url = initialData
@@ -208,60 +233,6 @@ export function BannerForm({ initialData }: BannerFormProps) {
                             )}
                         />
 
-                        <div className="grid grid-cols-2 gap-4">
-                            <FormField
-                                control={form.control}
-                                name="width"
-                                render={({ field }) => (
-                                    <FormItem>
-                                        <FormLabel>Ancho (px)</FormLabel>
-                                        <FormControl>
-                                            <Input type="number" {...field} />
-                                        </FormControl>
-                                        <FormMessage />
-                                    </FormItem>
-                                )}
-                            />
-                            <FormField
-                                control={form.control}
-                                name="height"
-                                render={({ field }) => (
-                                    <FormItem>
-                                        <FormLabel>Alto (px)</FormLabel>
-                                        <FormControl>
-                                            <Input type="number" {...field} />
-                                        </FormControl>
-                                        <FormMessage />
-                                    </FormItem>
-                                )}
-                            />
-                        </div>
-
-                        <FormField
-                            control={form.control}
-                            name="position"
-                            render={({ field }) => (
-                                <FormItem>
-                                    <FormLabel>Posición</FormLabel>
-                                    <Select onValueChange={field.onChange} value={field.value}>
-                                        <FormControl>
-                                            <SelectTrigger>
-                                                <SelectValue placeholder="Selecciona posición" />
-                                            </SelectTrigger>
-                                        </FormControl>
-                                        <SelectContent>
-                                            <SelectItem value="top">Top / Header</SelectItem>
-                                            <SelectItem value="sidebar">Barra Lateral</SelectItem>
-                                            <SelectItem value="footer">Pie de Página</SelectItem>
-                                            <SelectItem value="content">Dentro del Contenido</SelectItem>
-                                            <SelectItem value="popup">Popup / Modal</SelectItem>
-                                        </SelectContent>
-                                    </Select>
-                                    <FormMessage />
-                                </FormItem>
-                            )}
-                        />
-
                         <FormField
                             control={form.control}
                             name="is_global"
@@ -304,6 +275,50 @@ export function BannerForm({ initialData }: BannerFormProps) {
                             />
                         )}
 
+                        <FormField
+                            control={form.control}
+                            name="slot_id"
+                            render={({ field }) => (
+                                <FormItem>
+                                    <FormLabel>Hueco publicitario</FormLabel>
+                                    <Select value={field.value} onValueChange={field.onChange}
+                                        disabled={!targetingReady || !availableSlots.length}>
+                                        <FormControl><SelectTrigger><SelectValue placeholder="Selecciona un hueco" /></SelectTrigger></FormControl>
+                                        <SelectContent>
+                                            {availableSlots.map(slot => <SelectItem key={slot.slot_id} value={slot.slot_id}>{slot.label}</SelectItem>)}
+                                        </SelectContent>
+                                    </Select>
+                                    <FormDescription>
+                                        {slotsError ? "No se pudo cargar el catálogo de huecos." : !targetingReady ? "Cargando huecos…" : !domains.length ? "Selecciona los portales para ver los huecos disponibles." : !availableSlots.length ? "No hay huecos habilitados comunes para estos portales." : "Solo se muestran huecos habilitados en todos los portales seleccionados."}
+                                    </FormDescription>
+                                    <FormMessage />
+                                </FormItem>
+                            )}
+                        />
+                        <FormField
+                            control={form.control}
+                            name="width"
+                            render={() => (
+                                <FormItem>
+                                    <FormLabel>Tamaño permitido</FormLabel>
+                                    <Select value={selectedSize ? sizeKey(selectedSize) : ""} disabled={!selectedSlot}
+                                        onValueChange={value => {
+                                            const size = selectedSlot?.sizes.find(size => sizeKey(size) === value);
+                                            if (!size) return;
+                                            form.setValue("width", size.width, { shouldDirty: true, shouldValidate: true });
+                                            form.setValue("height", size.height, { shouldDirty: true, shouldValidate: true });
+                                            form.setValue("size_label", size.label, { shouldDirty: true });
+                                        }}>
+                                        <FormControl><SelectTrigger><SelectValue placeholder="Selecciona un tamaño" /></SelectTrigger></FormControl>
+                                        <SelectContent>
+                                            {selectedSlot?.sizes.map(size => <SelectItem key={sizeKey(size)} value={sizeKey(size)}>{size.label} px</SelectItem>)}
+                                        </SelectContent>
+                                    </Select>
+                                    <FormMessage />
+                                </FormItem>
+                            )}
+                        />
+
                         <FormField
                             control={form.control}
                             name="is_active"
@@ -414,7 +429,7 @@ export function BannerForm({ initialData }: BannerFormProps) {
                 </div>
 
                 <div className="flex gap-4">
-                    <Button type="submit" disabled={isSubmitting || uploading}>
+                    <Button type="submit" disabled={isSubmitting || uploading || !targetingReady || !selectedSlot || !selectedSize}>
                         {isSubmitting && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
                         {initialData ? "Actualizar Banner" : "Crear Banner"}
                     </Button>
diff --git a/frontend/src/components/ui/multi-select.tsx b/frontend/src/components/ui/multi-select.tsx
index 4a6b02b..f491c07 100644
--- a/frontend/src/components/ui/multi-select.tsx
+++ b/frontend/src/components/ui/multi-select.tsx
@@ -80,6 +80,8 @@ export function MultiSelect({
                             <Badge key={item} variant="secondary">
                                 {option ? option.label : item}
                                 <button
+                                    type="button"
+                                    aria-label={`Quitar ${option?.label || item}`}
                                     className="ml-1 ring-offset-background rounded-full outline-none focus:ring-2 focus:ring-ring focus:ring-offset-2"
                                     onKeyDown={(e) => {
                                         if (e.key === "Enter") {
diff --git a/frontend/src/lib/ad-slots.ts b/frontend/src/lib/ad-slots.ts
new file mode 100644
index 0000000..b48b28b
--- /dev/null
+++ b/frontend/src/lib/ad-slots.ts
@@ -0,0 +1,36 @@
+export interface SlotSize {
+    width: number
+    height: number
+    label: string
+}
+
+export interface AdSlot {
+    site_domain: string
+    slot_id: string
+    label: string
+    page: "home" | "article" | "global"
+    desktop_sizes: SlotSize[]
+    mobile_sizes: SlotSize[]
+    enabled: boolean
+}
+
+export const normalizeSiteDomain = (domain: string) =>
+    domain.trim().toLowerCase().replace(/\.$/, "").replace(/^www\./, "")
+
+export const sizeKey = (size: Pick<SlotSize, "width" | "height">) => `${size.width}x${size.height}`
+
+// A dimension pair may be desktop on one portal and mobile on another,
+// matching the API's combined desktop/mobile size validation.
+export function sharedSlots(catalog: AdSlot[], domains: string[]): (AdSlot & { sizes: SlotSize[] })[] {
+    const targets = [...new Set(domains.map(normalizeSiteDomain))]
+    if (!targets.length) return []
+    return catalog.filter(slot => slot.enabled && slot.site_domain === targets[0]).flatMap(slot => {
+        const entries = targets.map(domain => catalog.find(entry =>
+            entry.enabled && entry.site_domain === domain && entry.slot_id === slot.slot_id))
+        if (entries.some(entry => !entry)) return []
+        const sizes = [...new Map([...slot.desktop_sizes, ...slot.mobile_sizes]
+            .map(size => [sizeKey(size), size])).values()].filter(size => entries.every(entry =>
+            [...entry!.desktop_sizes, ...entry!.mobile_sizes].some(other => sizeKey(other) === sizeKey(size))))
+        return sizes.length ? [{ ...slot, sizes }] : []
+    })
+}
```

### H1.4 — `e8e4a4d`: feat(ads): add slot inventory and booking availability dashboard

```diff
diff --git a/backend/ads/api.py b/backend/ads/api.py
index 627382b..7802728 100644
--- a/backend/ads/api.py
+++ b/backend/ads/api.py
@@ -1,3 +1,5 @@
+from datetime import date
+
 from django.shortcuts import get_object_or_404
 from ninja import Router
 from ninja.errors import HttpError
@@ -10,11 +12,12 @@ from publishing.models import Site
 from .models import AdCampaign, AdBanner
 from .schemas import (
     AdCampaignCreate, AdCampaignUpdate, AdCampaignOut,
-    AdBannerCreate, AdBannerUpdate, AdBannerOut, AdSlotOut
+    AdBannerCreate, AdBannerUpdate, AdBannerOut, AdSlotOut, AdInventoryOut
 )
 
 from .slots import PORTAL_DOMAINS, SLOT_CATALOG
 from .slot_validation import validate_slot_targeting
+from .inventory import inventory_for_range
 
 router = Router(tags=["Advertising"])
 
@@ -60,6 +63,18 @@ def list_slots(request):
     return SLOT_CATALOG
 
 
+@router.get("/inventory", response=List[AdInventoryOut])
+def list_inventory(request, start_date: date = None, end_date: date = None):
+    start_date = start_date or timezone.now().date()
+    end_date = end_date or start_date
+    if start_date > end_date:
+        raise HttpError(422, "La fecha de inicio debe ser anterior o igual a la fecha de fin.")
+    # The exclusive upper bound must be representable by datetime.
+    if end_date == date.max:
+        raise HttpError(422, "La fecha de fin está fuera del rango permitido.")
+    return inventory_for_range(start_date, end_date)
+
+
 # --- Campaign Endpoints ---
 
 @router.get("/campaigns", response=List[AdCampaignOut])
diff --git a/backend/ads/inventory.py b/backend/ads/inventory.py
new file mode 100644
index 0000000..1b6d0f4
--- /dev/null
+++ b/backend/ads/inventory.py
@@ -0,0 +1,37 @@
+"""Read-only slot availability for an inclusive calendar-date range."""
+from datetime import datetime, time, timedelta, timezone as datetime_timezone
+
+
+from .models import AdBanner
+from .slots import SLOT_CATALOG
+from .slot_validation import normalize_site_domain
+
+
+def inventory_for_range(start_date, end_date):
+    start = datetime.combine(start_date, time.min, tzinfo=datetime_timezone.utc)
+    stop = datetime.combine(end_date + timedelta(days=1), time.min, tzinfo=datetime_timezone.utc)
+    banners = AdBanner.objects.filter(
+        is_active=True, campaign__is_active=True,
+        campaign__start_date__lt=stop, campaign__end_date__gte=start,
+    ).exclude(slot_id="").select_related("campaign").prefetch_related("target_sites")
+    by_slot = {}
+    for banner in banners:
+        domains = None if banner.is_global else {
+            normalize_site_domain(site.domain) for site in banner.target_sites.all()
+        }
+        booking = {
+            "banner_id": banner.id, "banner_name": banner.name,
+            "campaign_id": banner.campaign_id, "campaign_name": banner.campaign.name,
+            "start_date": banner.campaign.start_date, "end_date": banner.campaign.end_date,
+            "width": banner.width, "height": banner.height,
+        }
+        by_slot.setdefault(banner.slot_id, []).append((domains, booking))
+
+    rows = []
+    for slot in SLOT_CATALOG:
+        if not slot["enabled"]:
+            continue
+        bookings = [booking for domains, booking in by_slot.get(slot["slot_id"], [])
+                    if domains is None or slot["site_domain"] in domains]
+        rows.append({**slot, "status": "booked" if bookings else "free", "bookings": bookings})
+    return rows
diff --git a/backend/ads/schemas.py b/backend/ads/schemas.py
index d5acc12..fd6af1d 100644
--- a/backend/ads/schemas.py
+++ b/backend/ads/schemas.py
@@ -120,3 +120,19 @@ class AdSlotOut(Schema):
     desktop_sizes: List[AdSlotSizeOut]
     mobile_sizes: List[AdSlotSizeOut]
     enabled: bool
+
+
+class AdInventoryBookingOut(Schema):
+    banner_id: UUID
+    banner_name: str
+    campaign_id: UUID
+    campaign_name: str
+    start_date: datetime
+    end_date: datetime
+    width: int
+    height: int
+
+
+class AdInventoryOut(AdSlotOut):
+    status: Literal["free", "booked"]
+    bookings: List[AdInventoryBookingOut]
diff --git a/backend/ads/test_inventory.py b/backend/ads/test_inventory.py
new file mode 100644
index 0000000..da2acce
--- /dev/null
+++ b/backend/ads/test_inventory.py
@@ -0,0 +1,143 @@
+"""Authenticated, read-only inventory contracts. Run only with isolated settings."""
+from datetime import datetime, timedelta, timezone as dt_timezone
+from unittest.mock import patch
+
+from django.contrib.auth.models import User
+from django.test import Client, TestCase
+
+from ads.inventory import inventory_for_range
+from ads.models import AdBanner, AdCampaign
+from ads.slots import SLOT_CATALOG
+from core.models import Brand, MediaAsset, Organization
+from publishing.models import Site
+
+
+class InventoryApiTests(TestCase):
+    @classmethod
+    def setUpTestData(cls):
+        cls.org = Organization.objects.create(name="Sales", tax_id="INVENTORY", billing_email="sales@example.com")
+        cls.user = User.objects.create_user(username="inventory-sales")
+        cls.brand = Brand.objects.create(name="Advertiser", organization=cls.org)
+        cls.image = MediaAsset.objects.create(file="inventory-test.gif")
+        cls.clave = Site.objects.create(organization=cls.org, name="La Clave", domain="laclavenoticias.es")
+        cls.objective = Site.objects.create(organization=cls.org, name="El Objetivo", domain="elobjetivonoticias.es")
+        cls.start = datetime(2026, 10, 22, tzinfo=dt_timezone.utc)
+
+    def setUp(self):
+        self.client = Client(enforce_csrf_checks=True)
+        self.client.force_login(self.user)
+
+    def inventory(self, start="2026-10-22", end="2026-10-22"):
+        return self.client.get("/api/v1/ads/inventory", {"start_date": start, "end_date": end})
+
+    def row(self, rows, domain="laclavenoticias.es", slot="home-sidebar-top"):
+        return next(row for row in rows if row["site_domain"] == domain and row["slot_id"] == slot)
+
+    def banner(self, start=None, end=None, active=True, campaign_active=True, global_banner=False, site=None, slot="home-sidebar-top"):
+        campaign = AdCampaign.objects.create(brand=self.brand, name="Booked campaign", start_date=start or self.start,
+            end_date=end or self.start + timedelta(days=1), is_active=campaign_active)
+        banner = AdBanner.objects.create(campaign=campaign, image=self.image, name="Booked creative", slot_id=slot,
+            is_active=active, is_global=global_banner)
+        if not global_banner:
+            banner.target_sites.add(site or self.clave)
+        return banner
+
+    def test_authentication_required(self):
+        self.assertEqual(Client().get("/api/v1/ads/inventory").status_code, 401)
+
+    def test_empty_inventory_lists_only_enabled_slots_as_free(self):
+        response = self.inventory()
+        self.assertEqual(response.status_code, 200)
+        rows = response.json()
+        expected = {(slot["site_domain"], slot["slot_id"]) for slot in SLOT_CATALOG if slot["enabled"]}
+        self.assertEqual({(row["site_domain"], row["slot_id"]) for row in rows}, expected)
+        for row in rows:
+            self.assertEqual(row["status"], "free")
+            self.assertEqual(row["bookings"], [])
+            self.assertTrue(row["desktop_sizes"])
+
+    def test_targeted_booking_and_edit_identifiers(self):
+        banner = self.banner()
+        response = self.inventory()
+        self.assertEqual(response.status_code, 200)
+        row = self.row(response.json())
+        self.assertEqual(row["status"], "booked")
+        booking = row["bookings"][0]
+        self.assertEqual(booking["banner_id"], str(banner.id))
+        self.assertEqual(booking["campaign_id"], str(banner.campaign_id))
+        self.assertEqual(booking["campaign_name"], "Booked campaign")
+        self.assertEqual(booking["banner_name"], "Booked creative")
+        self.assertEqual((booking["width"], booking["height"]), (300, 250))
+        self.assertEqual(self.row(response.json(), "elobjetivonoticias.es")["status"], "free")
+        self.assertEqual(AdBanner.objects.count(), 1)
+        self.assertEqual(AdCampaign.objects.count(), 1)
+
+    def test_global_booking_marks_all_three_portals(self):
+        self.banner(global_banner=True)
+        rows = self.inventory().json()
+        booked = [row for row in rows if row["status"] == "booked"]
+        self.assertEqual(len(booked), 3)
+        self.assertTrue(all(row["slot_id"] == "home-sidebar-top" for row in booked))
+
+    def test_paused_campaign_and_banner_do_not_book_slots(self):
+        self.banner(campaign_active=False)
+        self.banner(active=False)
+        self.assertEqual(self.row(self.inventory().json())["status"], "free")
+
+    def test_non_overlapping_campaigns_are_free(self):
+        self.banner(start=self.start - timedelta(days=2), end=self.start - timedelta(microseconds=1))
+        self.banner(start=self.start + timedelta(days=1), end=self.start + timedelta(days=2))
+        self.assertEqual(self.row(self.inventory().json())["status"], "free")
+
+    def test_inclusive_day_boundaries_and_future_range(self):
+        self.banner(start=self.start - timedelta(days=1), end=self.start)
+        self.banner(start=self.start + timedelta(hours=23, minutes=59), end=self.start + timedelta(days=1))
+        self.assertEqual(len(self.row(self.inventory().json())["bookings"]), 2)
+        self.assertEqual(self.row(self.inventory("2026-11-01", "2026-11-10").json())["status"], "free")
+        self.banner(start=self.start + timedelta(days=10), end=self.start + timedelta(days=15))
+        self.assertEqual(self.row(self.inventory("2026-11-01", "2026-11-10").json())["status"], "booked")
+
+    def test_multi_day_range_includes_partial_overlap(self):
+        self.banner(start=self.start + timedelta(days=3), end=self.start + timedelta(days=4))
+        self.assertEqual(self.row(self.inventory("2026-10-22", "2026-10-30").json())["status"], "booked")
+
+    def test_multiple_creatives_are_listed(self):
+        first = self.banner()
+        second = self.banner()
+        row = self.row(self.inventory().json())
+        self.assertEqual(row["status"], "booked")
+        self.assertEqual({booking["banner_id"] for booking in row["bookings"]}, {str(first.id), str(second.id)})
+
+    def test_www_alias_and_multiple_target_portals(self):
+        self.clave.domain = "www.laclavenoticias.es"
+        self.clave.save()
+        banner = self.banner()
+        banner.target_sites.add(self.objective)
+        rows = self.inventory().json()
+        self.assertEqual(self.row(rows)["status"], "booked")
+        self.assertEqual(self.row(rows, "elobjetivonoticias.es")["status"], "booked")
+        self.assertEqual(self.row(rows, "eltitulardemurcia.es")["status"], "free")
+
+    def test_disabled_or_unassigned_slots_are_not_counted(self):
+        self.banner(slot="home-sidebar-middle")
+        self.banner(slot="")
+        self.assertTrue(all(row["status"] == "free" for row in self.inventory().json()))
+
+    def test_invalid_date_ranges_return_422(self):
+        for start, end in [("bad", "2026-10-22"), ("2026-10-23", "2026-10-22"), ("2026-10-22", "9999-12-31")]:
+            with self.subTest(start=start, end=end):
+                self.assertEqual(self.inventory(start, end).status_code, 422)
+
+    def test_default_range_is_today_utc(self):
+        self.banner()
+        with patch("ads.api.timezone.now", return_value=self.start):
+            response = self.client.get("/api/v1/ads/inventory")
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(self.row(response.json())["status"], "booked")
+
+    def test_inventory_avoids_queries_per_slot_or_booking(self):
+        for _ in range(3):
+            self.banner()
+        with self.assertNumQueries(2):
+            rows = inventory_for_range(self.start.date(), self.start.date())
+        self.assertEqual(len(self.row(rows)["bookings"]), 3)
diff --git a/frontend/src/app/dashboard/ads/inventory/page.tsx b/frontend/src/app/dashboard/ads/inventory/page.tsx
new file mode 100644
index 0000000..b9e4c18
--- /dev/null
+++ b/frontend/src/app/dashboard/ads/inventory/page.tsx
@@ -0,0 +1,111 @@
+"use client"
+
+import { useState, type FormEvent } from "react"
+import Link from "next/link"
+import useSWR from "swr"
+import { fetcher } from "@/lib/fetcher"
+import type { AdSlot } from "@/lib/ad-slots"
+import { Button } from "@/components/ui/button"
+import { Input } from "@/components/ui/input"
+import { Label } from "@/components/ui/label"
+import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table"
+
+interface Booking {
+    banner_id: string
+    banner_name: string
+    campaign_id: string
+    campaign_name: string
+    start_date: string
+    end_date: string
+    width: number
+    height: number
+}
+interface InventorySlot extends AdSlot {
+    status: "free" | "booked"
+    bookings: Booking[]
+}
+
+const dateTime = (value: string) => new Intl.DateTimeFormat("es-ES", {
+    dateStyle: "short", timeStyle: "short", timeZone: "UTC",
+}).format(new Date(value))
+
+export default function InventoryPage() {
+    const [range, setRange] = useState(() => {
+        const today = new Date().toISOString().slice(0, 10)
+        return { start: today, end: today }
+    })
+    const [draft, setDraft] = useState(range)
+    const [dateError, setDateError] = useState("")
+    const [portal, setPortal] = useState("")
+    const [status, setStatus] = useState("")
+    const query = new URLSearchParams({ start_date: range.start, end_date: range.end })
+    const { data, error, isLoading, isValidating, mutate } = useSWR<InventorySlot[]>(
+        `/api/v1/ads/inventory?${query}`, fetcher, { refreshInterval: 60000 }
+    )
+    const portals = [...new Set(data?.map(slot => slot.site_domain) || [])]
+    const rows = data?.filter(slot => (!portal || slot.site_domain === portal) && (!status || slot.status === status)) || []
+    const free = rows.filter(slot => slot.status === "free").length
+    const booked = rows.length - free
+
+    function applyRange(event: FormEvent<HTMLFormElement>) {
+        event.preventDefault()
+        if (!draft.start || !draft.end || draft.start > draft.end || draft.end === "9999-12-31") {
+            setDateError("Selecciona fechas válidas, con el inicio anterior o igual al fin.")
+            return
+        }
+        setDateError("")
+        setRange({ ...draft })
+    }
+
+    return (
+        <div className="flex flex-1 flex-col gap-6 p-4 md:p-6">
+            <div className="flex flex-wrap items-center justify-between gap-3">
+                <div>
+                    <h1 className="text-2xl font-bold">Inventario publicitario</h1>
+                    <p className="text-muted-foreground">Consulta los huecos libres y ocupados en cada portal.</p>
+                </div>
+                <Button type="button" variant="outline" disabled={isValidating} onClick={() => mutate()}>
+                    {isValidating ? "Actualizando..." : "Actualizar"}
+                </Button>
+            </div>
+            <form onSubmit={applyRange} className="flex flex-wrap items-end gap-3 rounded-md border p-4">
+                <div className="grid gap-2"><Label htmlFor="inventory-start">Desde (UTC)</Label>
+                    <Input id="inventory-start" type="date" required value={draft.start} onChange={event => setDraft({ ...draft, start: event.target.value })} /></div>
+                <div className="grid gap-2"><Label htmlFor="inventory-end">Hasta (UTC)</Label>
+                    <Input id="inventory-end" type="date" required value={draft.end} onChange={event => setDraft({ ...draft, end: event.target.value })} /></div>
+                <Button type="submit">Consultar</Button>
+            </form>
+            {dateError && <p role="alert" className="text-destructive">{dateError}</p>}
+            <div className="flex flex-wrap gap-4">
+                <div className="grid gap-2"><Label htmlFor="inventory-portal">Portal</Label>
+                    <select id="inventory-portal" className="h-9 rounded-md border bg-background px-3 text-sm" value={portal} onChange={event => setPortal(event.target.value)}>
+                        <option value="">Todos los portales</option>{portals.map(domain => <option key={domain} value={domain}>{domain}</option>)}
+                    </select></div>
+                <div className="grid gap-2"><Label htmlFor="inventory-status">Disponibilidad</Label>
+                    <select id="inventory-status" className="h-9 rounded-md border bg-background px-3 text-sm" value={status} onChange={event => setStatus(event.target.value)}>
+                        <option value="">Todos los estados</option><option value="free">Libre</option><option value="booked">Ocupado</option>
+                    </select></div>
+            </div>
+            <p className="text-sm text-muted-foreground">Rango consultado: {range.start} — {range.end}, ambos días incluidos en UTC. Un hueco está ocupado si un banner y su campaña están activos y sus fechas se solapan con el rango. Esta vista no bloquea nuevas reservas.</p>
+            {error ? <div role="alert">No se pudo cargar el inventario. <Button type="button" variant="outline" onClick={() => mutate()}>Reintentar</Button></div> : isLoading ?
+                <p role="status">Cargando inventario...</p> : <>
+                    <p role="status" className="text-sm">{free} libres · {booked} ocupados · {rows.length} huecos</p>
+                    <div className="overflow-x-auto rounded-md border"><Table>
+                        <TableHeader><TableRow><TableHead>Portal</TableHead><TableHead>Hueco</TableHead><TableHead>Tamaños permitidos</TableHead><TableHead>Estado</TableHead><TableHead>Campañas y banners</TableHead></TableRow></TableHeader>
+                        <TableBody>{!rows.length ? <TableRow><TableCell colSpan={5}>No hay huecos que coincidan con los filtros.</TableCell></TableRow> : rows.map(slot => <TableRow key={`${slot.site_domain}/${slot.slot_id}`}>
+                            <TableCell>{slot.site_domain}</TableCell>
+                            <TableCell><p className="font-medium">{slot.label}</p><p className="text-xs text-muted-foreground">{slot.slot_id}</p></TableCell>
+                            <TableCell><p>Escritorio: {slot.desktop_sizes.map(size => size.label).join(", ")} px</p><p className="text-xs text-muted-foreground">Móvil: {slot.mobile_sizes.length ? slot.mobile_sizes.map(size => size.label).join(", ") + " px" : "No disponible"}</p></TableCell>
+                            <TableCell><span className={slot.status === "booked" ? "rounded-full bg-amber-100 px-2 py-1 text-xs font-medium text-amber-900" : "rounded-full bg-green-100 px-2 py-1 text-xs font-medium text-green-900"}>{slot.status === "booked" ? "Ocupado" : "Libre"}</span></TableCell>
+                            <TableCell>{slot.bookings.length ? <ul className="space-y-3">{slot.bookings.map(booking => <li key={booking.banner_id}>
+                                <Link className="font-medium underline" href={`/dashboard/ads/campaigns/${booking.campaign_id}`}>{booking.campaign_name}</Link>
+                                <p><Link className="underline" href={`/dashboard/ads/banners/${booking.banner_id}`}>{booking.banner_name}</Link> · {booking.width}×{booking.height}</p>
+                                <p className="text-xs text-muted-foreground">{dateTime(booking.start_date)} — {dateTime(booking.end_date)} UTC</p>
+                            </li>)}</ul> : "Sin campañas activas en este rango"}</TableCell>
+                        </TableRow>)}</TableBody>
+                    </Table></div>
+                </>}
+            <p className="text-xs text-muted-foreground">Solo se incluyen huecos habilitados. Los banners antiguos sin hueco asignado deben actualizarse antes de aparecer en el inventario.</p>
+        </div>
+    )
+}
diff --git a/frontend/src/components/app-sidebar.tsx b/frontend/src/components/app-sidebar.tsx
index fe46419..bdf4d87 100644
--- a/frontend/src/components/app-sidebar.tsx
+++ b/frontend/src/components/app-sidebar.tsx
@@ -96,6 +96,11 @@ const data = {
     }
   ],
   navAdvertising: [
+    {
+      title: "Inventario",
+      url: "/dashboard/ads/inventory",
+      icon: IconReport,
+    },
     {
       title: "Anunciantes",
       url: "/dashboard/ads/advertisers",
```
