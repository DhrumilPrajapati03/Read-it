# H0 advertising dashboard — implementation record

Project: **PROJ-0075 M4B Newspaper**  
Assignee: **Dhrumil**  
Prepared: **14 October 2026**  
Scope: committed H0.1–H0.5 implementation, ending at `67d2430`.

H0 makes advertiser, campaign and image-banner management usable in the sales dashboard. Code and contract tests have been committed; backend test execution and browser/live-domain acceptance remain outstanding. Live banner delivery belongs to H2.

## 1. Commits and branches

| Ticket | Commit | Feature branch | Change |
|---|---|---|---|
| H0.1 | `59c919b` | `feature/h0-1-advertiser-crud` | Advertiser CRUD and dashboard management |
| H0.2 | `26b9471` | `feature/h0-2-edit-params` | Unwrap edit-page params for Next 16 |
| H0.3 | `3e7461f` | `feature/h0-3-form-validation` | Shared CSRF requests and form/API validation |
| H0.4 | `4ce760d` | `feature/h0-4-advertiser-required` | Link empty campaign form to advertiser creation |
| H0.5 | `67d2430` | `feature/h0-5-authenticated-contract-tests` | Replace anonymous ads tests with authenticated contracts |

The branches were created in sequence from the preceding H0 branch. These commits were created locally; this work did not push them or verify their deployment.

## 2. Changed files

| File | Milestone | Specific changed part and purpose |
|---|---|---|
| `backend/core/api/schemas.py` | H0.1 | Added Pydantic imports and `BrandWriteSchema` to validate advertiser names. |
| `backend/core/api/views.py` | H0.1 | Added organization lookup; scoped advertiser listing; added create, update and guarded delete endpoints. |
| `backend/core/tests.py` | H0.1 | Replaced placeholder with 11 authenticated advertiser API tests. |
| `frontend/src/app/dashboard/ads/advertisers/page.tsx` | H0.1, H0.3 | New advertiser management page; later replaced its local mutation helper with shared `apiFetch`. |
| `frontend/src/components/app-sidebar.tsx` | H0.1 | Added “Anunciantes” to `navAdvertising`. |
| `frontend/src/app/dashboard/ads/campaigns/[id]/page.tsx` | H0.2 | Imported React `use`, typed `params` as a promise, resolved `id` before fetching. |
| `frontend/src/app/dashboard/ads/banners/[id]/page.tsx` | H0.2 | Applied the same promise-params fix to banner editing. |
| `frontend/src/lib/api-fetch.ts` | H0.3 | New shared authenticated request helper, CSRF handling and structured error handling. |
| `frontend/src/components/ads/campaign-form.tsx` | H0.3, H0.4 | Date-order and advertiser UUID validation; shared fetch helpers; save guard; empty-state link. |
| `frontend/src/components/ads/banner-form.tsx` | H0.3 | Campaign UUID and active-image validation; CSRF upload/save; disabled script input; visible image errors. |
| `frontend/src/app/dashboard/ads/campaigns/page.tsx` | H0.3 | Campaign deletion uses `apiFetch` and displays API error messages. |
| `frontend/src/app/dashboard/ads/banners/page.tsx` | H0.3 | Banner deletion uses `apiFetch` and displays API error messages. |
| `backend/ads/api.py` | H0.3 | Added campaign-date and banner validators; called them before create/update persistence. |
| `backend/ads/tests.py` | H0.5 | Replaced three anonymous tests with 14 authenticated contract tests. |

**14 unique files** are included in H0. Exact patches for every commit appear in section 9.

## 3. H0.1 — advertiser CRUD

### Name validation: `backend/core/api/schemas.py`

```python
class BrandWriteSchema(Schema):
    name: str = Field(min_length=1, max_length=100)

    @field_validator("name")
    @classmethod
    def validate_name(cls, value):
        value = value.strip()
        if not value:
            raise ValueError("El nombre del anunciante es obligatorio")
        return value
```

Names are required, trimmed and limited to 100 characters. Invalid input returns schema validation errors with HTTP 422.

### Organization assignment and endpoints: `backend/core/api/views.py`

`advertiser_organization(request)` resolves the user's profile organization. A superuser without one may use the single existing organization when exactly one exists; otherwise the request receives HTTP 403.

| Method | URL | Behavior |
|---|---|---|
| GET | `/api/v1/core/brands` | Lists only the user's organization advertisers, ordered by name and ID. |
| POST | `/api/v1/core/brands` | Creates an advertiser in the resolved organization; returns ID and name. |
| PUT | `/api/v1/core/brands/{brand_id}` | Renames an advertiser in that organization. |
| DELETE | `/api/v1/core/brands/{brand_id}` | Deletes only when no campaign is marked active. |

Updates and deletes return 404 for missing advertisers or advertisers from another organization. Successful mutations currently return HTTP 200.

Deletion checks happen before `.delete()` inside `transaction.atomic()`, with a row lock on the advertiser:

```python
with transaction.atomic():
    brand = get_object_or_404(
        Brand.objects.select_for_update(), id=brand_id, organization=organization
    )
    if brand.campaigns.filter(is_active=True).exists():
        raise HttpError(409, "No se puede eliminar un anunciante con campañas activas. Desactívalas primero.")
    brand.delete()
```

“Active” means the campaign's `is_active` flag, regardless of its dates. A blocked deletion preserves the advertiser, campaigns, banners and metrics. If all campaigns are inactive, deletion cascades to those campaigns, their banners and metrics. The UI confirmation explicitly explains this consequence.

### Advertiser page and navigation

`frontend/src/app/dashboard/ads/advertisers/page.tsx` adds:

- SWR listing and case-insensitive name search.
- A name form supporting create and edit, with cancel editing.
- Delete confirmation, busy-state controls and success toasts.
- Loading, empty, load-error/retry and action-error states.
- Required name input with `maxLength={100}`.

H0.1 initially used a local `advertiserMutation()` helper. H0.3 removed it and routed create/update/delete through `apiFetch`.

`frontend/src/components/app-sidebar.tsx` adds:

```tsx
{
    title: "Anunciantes",
    url: "/dashboard/ads/advertisers",
    icon: IconUsers,
}
```

### Advertiser tests: `backend/core/tests.py`

`AdvertiserApiTests` contains 11 tests for scoped listing, create/update, invalid names, cross-organization/missing IDs, blocked active-campaign deletion with no changes, deletion without campaigns, inactive-campaign cascade deletion, CSRF, anonymous rejection, missing organization, and single-organization superuser fallback.

## 4. H0.2 — repair campaign and banner editing

Both dynamic edit pages now import `use` from React and resolve Next 16 promise params:

```tsx
// Campaign edit page
export default function EditCampaignPage({ params }: { params: Promise<{ id: string }> }) {
    const { id } = use(params)
    const { data: campaign, error, isLoading } = useSWR(
        `/api/v1/ads/campaigns/${id}`,
        fetcher
    )
```

The banner page uses the same pattern with `EditBannerPage` and `/api/v1/ads/banners/${id}`. The previous synchronous `${params.id}` access was removed. Existing loading states and forms remain; the fetched record is still passed to `CampaignForm` or `BannerForm` as `initialData`.

## 5. H0.3 — shared requests and validation

### Shared helper: `frontend/src/lib/api-fetch.ts`

New functions/classes:

| Symbol | Purpose |
|---|---|
| `ApiError` | Carries an error message, HTTP `status`, and parsed response `info`. |
| `errorMessage()` | Uses string `detail` messages or joins field-level validation errors from a `detail` array; otherwise supplies a status-based fallback. |
| `checkedFetch()` | Sends `credentials: "include"`, checks `response.ok`, and throws `ApiError` on failed responses. |
| `apiFetch<T>()` | Handles CSRF, request headers and typed JSON results; returns `undefined` for HTTP 204. |

For mutations, the helper refreshes the CSRF cookie through authenticated `check-auth`, reads it, and sets the header:

```tsx
const method = (options.method || "GET").toUpperCase()
if (!["GET", "HEAD", "OPTIONS"].includes(method)) {
    await checkedFetch("/api/v1/core/check-auth", { method: "GET" })
    const token = document.cookie.split(";").map(cookie => cookie.trim())
        .find(cookie => cookie.startsWith("csrftoken="))?.slice("csrftoken=".length)
    if (!token) throw new Error("No se pudo obtener el token de seguridad. Recarga la página.")
    headers.set("X-CSRFToken", decodeURIComponent(token))
}
```

String request bodies receive `Content-Type: application/json` if not already supplied. `FormData` uploads retain the browser-generated multipart boundary. The existing `frontend/src/lib/fetcher.ts` was reused for SWR reads and was not modified.

### Campaign form: `frontend/src/components/ads/campaign-form.tsx`

Advertiser selection now requires a UUID. The Zod schema checks both dates:

```tsx
}).refine(values => values.start_date <= values.end_date, {
    message: "La fecha de inicio debe ser anterior o igual a la fecha de fin.",
    path: ["end_date"],
})
```

Saves use `apiFetch`; dates remain serialized with `toISOString()`. Errors are displayed through toasts. Brands use the existing shared SWR fetcher, so failed reads are treated as errors rather than usable JSON. The previous unused import and speculative advertiser-fetch comments were removed.

### Campaign API: `backend/ads/api.py`

`validate_campaign_dates(start, end)` rejects null dates, normalizes naive datetimes for comparison, and rejects `start > end` with HTTP 422. Equal dates are accepted.

Create validates the supplied dates before creating the record. Update combines supplied fields with the stored record before validation:

```python
data = payload.dict(exclude_unset=True)
validate_campaign_dates(
    data.get("start_date", campaign.start_date),
    data.get("end_date", campaign.end_date),
)
```

Changing just one date therefore cannot bypass the rule. Validation happens before saving changes.

### Banner form: `frontend/src/components/ads/banner-form.tsx`

The selected campaign must be a UUID. The form requires an image when active:

```tsx
}).refine(values => !values.is_active || !!values.image_id, {
    message: "Un banner activo requiere una imagen.",
    path: ["image_id"],
})
```

Image upload and banner save use `apiFetch`. Upload responses supply the image ID and preview URL. Image add/remove triggers validation; the creative section displays the image-field error. Save is disabled while submitting or uploading.

The script textarea is `disabled`, its description directs users to image creatives, its default is empty, and saved payloads explicitly set `script_code: ""`. Saving an existing script banner through this form clears its script value and requires an image if active.

### Banner API: `backend/ads/api.py`

`validate_banner(active, image_id, script_code)`:

- Rejects non-empty scripts with HTTP 422.
- Rejects active banners without an image ID with HTTP 422.
- Resolves supplied image IDs with `get_object_or_404(MediaAsset, ...)`.
- Rejects active banners whose referenced media has no file.

Create validates supplied values. Update validates the resulting banner state, retaining stored values for omitted fields:

```python
validate_banner(
    data.get("is_active", banner.is_active),
    data.get("image_id", banner.image_id),
    data.get("script_code", banner.script_code),
)
```

An inactive banner can remain without an image. Activating it without adding an image, or removing the image from an active banner, is rejected. Existing non-empty scripts must be cleared before an update can pass this validation. This checks the media reference/file field; it does not add image-content or slot-size validation.

### Delete actions

Both campaign and banner listing pages replace raw `fetch(..., { method: "DELETE" })` with:

```tsx
await apiFetch(`${API_URL}/${deleteId}`, { method: "DELETE" })
```

Their error toasts now display the thrown API message. Advertiser mutations also use the same helper. Upload backend behavior was not changed by H0; only the frontend upload request gained shared CSRF handling.

## 6. H0.4 — prevent saving without advertisers and link to creation

The save guard was introduced in H0.3:

```tsx
const canSave = !brandsLoading && !brandsError && !!brands?.length

async function onSubmit(values: z.infer<typeof formSchema>) {
    if (!canSave) return
    // Save through apiFetch...
}

<Button type="submit" disabled={isSubmitting || !canSave}>
```

Both campaign create and edit forms are blocked while advertisers load, if loading fails, or if none exist. This is a frontend guard; the API already requires an existing brand for campaign creation.

H0.4 adds `import Link from "next/link"` and replaces the empty-state message with:

```tsx
{brandsError ? (
    <p role="alert">No se pudieron cargar los anunciantes.</p>
) : !brandsLoading && !brands?.length && (
    <div role="alert" className="space-y-2">
        <p>Crea un anunciante antes de guardar una campaña.</p>
        <Button asChild variant="outline">
            <Link href="/dashboard/ads/advertisers">Crear anunciante</Link>
        </Button>
    </div>
)}
```

“Crear anunciante” opens the existing advertiser page containing its creation form.

## 7. H0.5 — authenticated contract tests

`backend/ads/tests.py` replaces three anonymous tests with `AdsApiContractTests`. Setup creates a user with an organization, advertiser, portal and media reference, logs in with `force_login`, obtains the CSRF cookie via `check-auth`, and uses `Client(enforce_csrf_checks=True)`.

```python
self.client = Client(enforce_csrf_checks=True)
self.client.force_login(self.user)
response = self.client.get("/api/v1/core/check-auth")
self.assertEqual(response.status_code, 200)
self.csrf = self.client.cookies["csrftoken"].value
```

JSON writes send `HTTP_X_CSRFTOKEN` unless the test intentionally omits it. File storage is overridden to `InMemoryStorage` so upload tests do not write production media.

| Test method | Contract covered |
|---|---|
| `test_campaign_crud` | Create, list, reopen, update name/budget while preserving dates, and delete. |
| `test_banner_crud_and_campaign_filter` | Create with image/portal, filter by campaign, edit, clear targeting, reopen and delete. |
| `test_anonymous_requests_rejected` | Anonymous collection/detail reads and all campaign/banner mutation methods return 401. |
| `test_mutations_require_csrf` | Missing CSRF returns 403 for create/update/delete and preserves records. |
| `test_invalid_csrf_rejected` | Invalid CSRF token returns 403 without creating a campaign. |
| `test_reversed_dates_rejected_on_create` | Reversed dates return 422 without creating a record. |
| `test_partial_date_updates_validate_resulting_campaign` | Invalid single-date, null-date and malformed-date updates return 422 and preserve stored dates. |
| `test_equal_dates_allowed_on_create_and_update` | Equal start/end dates are accepted. |
| `test_active_banner_requires_image` | Active creation without an image returns 422. |
| `test_banner_updates_validate_resulting_state` | Activation requires an image; removing an active image is rejected; deactivation and removal together succeed. |
| `test_inactive_banner_without_image_allowed` | Inactive image-free creation succeeds. |
| `test_scripts_rejected_on_create_and_update` | Non-empty scripts return 422 and existing data remains unchanged. |
| `test_missing_resources_rejected` | Missing campaign/banner detail/update/delete and missing brand/campaign/image references return 404. |
| `test_salesperson_flow_with_image_upload` | Advertiser → campaign → upload a GIF → image banner → reopen → edit → reopen with the image retained. |

There are **25 test methods** across the changed core and ads suites: 11 advertiser tests and 14 advertising contract tests. Test methods describe intended assertions; their presence is not evidence that execution passed.

## 8. Verification and remaining acceptance

| Item | Recorded status |
|---|---|
| H0.2 focused ESLint on both edit pages | Passed. |
| Git whitespace/diff checks | Passed for H0.2–H0.5 changes. |
| H0.5 Python syntax parse | Passed. |
| H0.3 frontend lint | Not run; command request was rejected. |
| `python manage.py test ads core` | Not successfully executed. System Python lacked Django; a later isolated environment setup request was rejected. |
| Browser: open existing campaign/banner UUID and save | Not verified. |
| Live-domain salesperson demo | Not performed. |
| Push/MR/deployment state | Not verified; the user retained responsibility for pushing. |

No tests were executed inside production containers. No production database was modified, and containers were not restarted, rebuilt, stopped or changed. Temporary test dependencies were installed under `/tmp`, but no completed isolated test run exists.

Future test execution must use a separate isolated database, no production credentials and no external-service access. In-memory file storage in the test class does **not** itself isolate database settings; never run this suite with production configuration.

H0 is not accepted as complete until tests pass and a salesperson can create/edit an advertiser, campaign and image banner without developer help. The live banner delivery, real slot catalog, measurement, reporting, and operations milestones remain outside these changes. SaaS, Stripe, multi-tenancy implementation and Frappe integration remain deferred; advertiser organization assignment does not add full campaign/banner organization isolation.

Existing unrelated working-tree changes—including media optimization in `backend/core/api/views.py`, Compose, article/account UI and portal changes—are excluded from this record. No H0 database migration, dependency-file change, deployment configuration change or portal delivery change was committed.

## 9. Exact committed code patches

The following patches are copied directly from Git. They preserve all changes in each milestone, including code later replaced by another H0 commit. The descriptions above explain the final combined behavior.

### H0.1 — `59c919b`: feat(ads): add advertiser CRUD and dashboard management

```diff
diff --git a/backend/core/api/schemas.py b/backend/core/api/schemas.py
index 45e6185..1bb7750 100644
--- a/backend/core/api/schemas.py
+++ b/backend/core/api/schemas.py
@@ -1,4 +1,5 @@
 from ninja import Schema, ModelSchema
+from pydantic import Field, field_validator
 from typing import List, Optional
 from uuid import UUID
 from datetime import datetime
@@ -16,6 +17,18 @@ class BrandSchema(ModelSchema):
         model = Brand
         fields = ['id', 'name']
 
+class BrandWriteSchema(Schema):
+    name: str = Field(min_length=1, max_length=100)
+
+    @field_validator("name")
+    @classmethod
+    def validate_name(cls, value):
+        value = value.strip()
+        if not value:
+            raise ValueError("El nombre del anunciante es obligatorio")
+        return value
+
+
 class AddressCreateSchema(Schema):
     organization_id: Optional[UUID] = None # Optional: If supplied (by admin), uses it. Else uses current org.
     address_type: str = 'other'
diff --git a/backend/core/api/views.py b/backend/core/api/views.py
index ca33046..eb04edc 100644
--- a/backend/core/api/views.py
+++ b/backend/core/api/views.py
@@ -4,6 +4,7 @@ from datetime import datetime
 from uuid import UUID
 from pathlib import Path
 from django.shortcuts import get_object_or_404
+from django.db import transaction
 from django.contrib.auth.models import User
 from core.models import Organization, Address, UserProfile, EmailServerConfig, Brand, MediaAsset
 from publishing.models import Topic, Tag
@@ -12,7 +13,7 @@ from .schemas import (
     AddressSchema, AddressCreateSchema, AddressUpdateSchema,
     UserSchema,
     EmailConfigSchema, EmailConfigCreateSchema, EmailConfigUpdateSchema,
-    BrandSchema
+    BrandSchema, BrandWriteSchema
 )
 
 router = Router()
@@ -99,9 +100,46 @@ def list_invoices(request):
     ]
 
 # --- Brand ---
+def advertiser_organization(request):
+    profile = UserProfile.objects.filter(user=request.user).first()
+    if profile and profile.organization_id:
+        return profile.organization
+    # Support the existing single-organization backoffice administrator.
+    if request.user.is_superuser and Organization.objects.count() == 1:
+        return Organization.objects.get()
+    raise HttpError(403, "El usuario no tiene una organización asignada")
+
+
 @router.get("/brands", response=List[BrandSchema])
 def list_brands(request):
-    return Brand.objects.all()
+    return Brand.objects.filter(organization=advertiser_organization(request)).order_by("name", "id")
+
+
+@router.post("/brands", response=BrandSchema)
+def create_brand(request, data: BrandWriteSchema):
+    return Brand.objects.create(organization=advertiser_organization(request), name=data.name)
+
+
+@router.put("/brands/{brand_id}", response=BrandSchema)
+def update_brand(request, brand_id: UUID, data: BrandWriteSchema):
+    brand = get_object_or_404(Brand, id=brand_id, organization=advertiser_organization(request))
+    brand.name = data.name
+    brand.save(update_fields=["name", "updated_at"])
+    return brand
+
+
+@router.delete("/brands/{brand_id}")
+def delete_brand(request, brand_id: UUID):
+    organization = advertiser_organization(request)
+    with transaction.atomic():
+        brand = get_object_or_404(
+            Brand.objects.select_for_update(), id=brand_id, organization=organization
+        )
+        if brand.campaigns.filter(is_active=True).exists():
+            raise HttpError(409, "No se puede eliminar un anunciante con campañas activas. Desactívalas primero.")
+        brand.delete()
+    return {"success": True}
+
 
 # --- Organization ---
 
diff --git a/backend/core/tests.py b/backend/core/tests.py
index 7ce503c..c57e1cd 100644
--- a/backend/core/tests.py
+++ b/backend/core/tests.py
@@ -1,3 +1,113 @@
-from django.test import TestCase
+from datetime import timedelta
+from uuid import uuid4
 
-# Create your tests here.
+from django.contrib.auth.models import User
+from django.test import Client, TestCase
+from django.utils import timezone
+
+from ads.models import AdBanner, AdCampaign, AdMetric
+from core.models import Brand, Organization, UserProfile
+from publishing.models import Site
+
+
+class AdvertiserApiTests(TestCase):
+    url = "/api/v1/core/brands"
+
+    def setUp(self):
+        self.org = Organization.objects.create(name="Sales", tax_id="SALES", billing_email="sales@example.com")
+        self.other_org = Organization.objects.create(name="Other", tax_id="OTHER", billing_email="other@example.com")
+        self.user = User.objects.create_user(username="sales", password="test-password")
+        UserProfile.objects.create(user=self.user, organization=self.org)
+        self.brand = Brand.objects.create(name="Advertiser", organization=self.org)
+        self.other_brand = Brand.objects.create(name="Other advertiser", organization=self.other_org)
+        self.client = Client(enforce_csrf_checks=True)
+        self.client.force_login(self.user)
+        self.client.get("/api/v1/core/check-auth")
+        self.headers = {"HTTP_X_CSRFTOKEN": self.client.cookies["csrftoken"].value}
+
+    def write(self, method, url, data=None, csrf=True):
+        return getattr(self.client, method)(url, data=data or {}, content_type="application/json", **(self.headers if csrf else {}))
+
+    def test_list_is_scoped(self):
+        response = self.client.get(self.url)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.json(), [{"id": str(self.brand.id), "name": self.brand.name}])
+
+    def test_create_and_update(self):
+        response = self.write("post", self.url, {"name": "  New advertiser  "})
+        self.assertEqual(response.status_code, 200)
+        brand = Brand.objects.get(id=response.json()["id"])
+        self.assertEqual(brand.organization, self.org)
+        self.assertEqual(brand.name, "New advertiser")
+        response = self.write("put", f"{self.url}/{brand.id}", {"name": "Renamed"})
+        self.assertEqual(response.status_code, 200)
+        brand.refresh_from_db()
+        self.assertEqual(brand.name, "Renamed")
+
+    def test_invalid_names(self):
+        for method, url in [("post", self.url), ("put", f"{self.url}/{self.brand.id}")]:
+            for data in [{}, {"name": ""}, {"name": "  "}, {"name": "x" * 101}, {"name": None}]:
+                with self.subTest(method=method, data=data):
+                    self.assertEqual(self.write(method, url, data).status_code, 422)
+        self.brand.refresh_from_db()
+        self.assertEqual(self.brand.name, "Advertiser")
+        self.assertEqual(Brand.objects.count(), 2)
+
+    def test_cannot_access_another_organization_or_missing_brand(self):
+        for brand_id in [self.other_brand.id, uuid4()]:
+            for method in ["put", "delete"]:
+                self.assertEqual(self.write(method, f"{self.url}/{brand_id}", {"name": "Changed"}).status_code, 404)
+        self.other_brand.refresh_from_db()
+        self.assertEqual(self.other_brand.name, "Other advertiser")
+
+    def campaign_tree(self, active):
+        campaign = AdCampaign.objects.create(brand=self.brand, name="Campaign", start_date=timezone.now(), end_date=timezone.now() + timedelta(days=30), is_active=active)
+        banner = AdBanner.objects.create(campaign=campaign)
+        site = Site.objects.create(name="Portal", domain="portal.example.com")
+        metric = AdMetric.objects.create(banner=banner, site=site, date=timezone.now().date(), impressions=7)
+        return campaign, banner, metric
+
+    def test_active_campaign_blocks_deletion_without_changes(self):
+        campaign, banner, metric = self.campaign_tree(True)
+        inactive = AdCampaign.objects.create(brand=self.brand, name="Paused", start_date=timezone.now(), end_date=timezone.now(), is_active=False)
+        response = self.write("delete", f"{self.url}/{self.brand.id}")
+        self.assertEqual(response.status_code, 409)
+        for obj in [self.brand, campaign, inactive, banner, metric]:
+            obj.refresh_from_db()
+        self.assertEqual(metric.impressions, 7)
+
+    def test_delete_with_no_campaigns(self):
+        self.assertEqual(self.write("delete", f"{self.url}/{self.brand.id}").status_code, 200)
+        self.assertFalse(Brand.objects.filter(id=self.brand.id).exists())
+
+    def test_delete_inactive_campaigns(self):
+        campaign, banner, metric = self.campaign_tree(False)
+        self.assertEqual(self.write("delete", f"{self.url}/{self.brand.id}").status_code, 200)
+        for obj in [self.brand, campaign, banner, metric]:
+            self.assertFalse(type(obj).objects.filter(pk=obj.pk).exists())
+        self.assertTrue(Brand.objects.filter(id=self.other_brand.id).exists())
+
+    def test_mutations_require_csrf(self):
+        for method, url in [("post", self.url), ("put", f"{self.url}/{self.brand.id}"), ("delete", f"{self.url}/{self.brand.id}")]:
+            self.assertEqual(self.write(method, url, {"name": "Changed"}, csrf=False).status_code, 403)
+        self.brand.refresh_from_db()
+        self.assertEqual(self.brand.name, "Advertiser")
+
+    def test_anonymous_access_rejected(self):
+        anonymous = Client()
+        self.assertEqual(anonymous.get(self.url).status_code, 401)
+        for method, url in [("post", self.url), ("put", f"{self.url}/{self.brand.id}"), ("delete", f"{self.url}/{self.brand.id}")]:
+            self.assertEqual(getattr(anonymous, method)(url, data={"name": "Changed"}, content_type="application/json").status_code, 401)
+
+    def test_missing_organization_rejected(self):
+        self.user.profile.organization = None
+        self.user.profile.save()
+        self.assertEqual(self.write("post", self.url, {"name": "Unassigned"}).status_code, 403)
+
+    def test_single_organization_superuser(self):
+        self.other_org.delete()
+        admin = User.objects.create_superuser(username="admin", password="test-password")
+        self.client.force_login(admin)
+        response = self.write("post", self.url, {"name": "Admin advertiser"})
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(Brand.objects.get(id=response.json()["id"]).organization, self.org)
diff --git a/frontend/src/app/dashboard/ads/advertisers/page.tsx b/frontend/src/app/dashboard/ads/advertisers/page.tsx
new file mode 100644
index 0000000..9b64339
--- /dev/null
+++ b/frontend/src/app/dashboard/ads/advertisers/page.tsx
@@ -0,0 +1,115 @@
+"use client"
+
+import { useState, type FormEvent } from "react"
+import useSWR from "swr"
+import { Pencil, Trash2 } from "lucide-react"
+import { toast } from "sonner"
+import { fetcher } from "@/lib/fetcher"
+import { Button } from "@/components/ui/button"
+import { Input } from "@/components/ui/input"
+import { Label } from "@/components/ui/label"
+import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table"
+import { AlertDialog, AlertDialogCancel, AlertDialogContent, AlertDialogDescription, AlertDialogFooter, AlertDialogHeader, AlertDialogTitle } from "@/components/ui/alert-dialog"
+
+interface Advertiser { id: string; name: string }
+const API_URL = "/api/v1/core/brands"
+
+async function advertiserMutation(url: string, method: string, name?: string) {
+    // Refresh the CSRF cookie for older sessions as well as newly logged-in users.
+    const auth = await fetch("/api/v1/core/check-auth", { credentials: "include" })
+    if (!auth.ok) throw new Error("La sesión ha caducado. Inicia sesión de nuevo.")
+    const token = document.cookie.split("; ").find(cookie => cookie.startsWith("csrftoken="))?.slice("csrftoken=".length)
+    if (!token) throw new Error("No se pudo obtener el token de seguridad. Recarga la página.")
+    const response = await fetch(url, {
+        method,
+        credentials: "include",
+        headers: { "Content-Type": "application/json", "X-CSRFToken": decodeURIComponent(token) },
+        ...(name !== undefined ? { body: JSON.stringify({ name }) } : {}),
+    })
+    if (!response.ok) {
+        const body = await response.json().catch(() => null)
+        throw new Error(typeof body?.detail === "string" ? body.detail : "No se pudo guardar el cambio. Comprueba el nombre e inténtalo de nuevo.")
+    }
+}
+
+export default function AdvertisersPage() {
+    const { data: advertisers, error, isLoading, mutate } = useSWR<Advertiser[]>(API_URL, fetcher)
+    const [name, setName] = useState("")
+    const [editingId, setEditingId] = useState<string | null>(null)
+    const [deleteTarget, setDeleteTarget] = useState<Advertiser | null>(null)
+    const [search, setSearch] = useState("")
+    const [busy, setBusy] = useState(false)
+    const [actionError, setActionError] = useState<string | null>(null)
+
+    function resetForm() { setName(""); setEditingId(null); setActionError(null) }
+
+    async function save(event: FormEvent<HTMLFormElement>) {
+        event.preventDefault()
+        if (busy || !name.trim()) return
+        setBusy(true)
+        setActionError(null)
+        try {
+            await advertiserMutation(editingId ? `${API_URL}/${editingId}` : API_URL, editingId ? "PUT" : "POST", name.trim())
+            toast.success(editingId ? "Anunciante actualizado" : "Anunciante creado")
+            resetForm()
+            await mutate()
+        } catch (error) {
+            setActionError(error instanceof Error ? error.message : "No se pudo guardar el anunciante")
+        } finally { setBusy(false) }
+    }
+
+    async function remove() {
+        if (!deleteTarget || busy) return
+        setBusy(true)
+        setActionError(null)
+        try {
+            await advertiserMutation(`${API_URL}/${deleteTarget.id}`, "DELETE")
+            if (editingId === deleteTarget.id) resetForm()
+            setDeleteTarget(null)
+            toast.success("Anunciante eliminado")
+            await mutate()
+        } catch (error) {
+            setDeleteTarget(null)
+            setActionError(error instanceof Error ? error.message : "No se pudo eliminar el anunciante")
+        } finally { setBusy(false) }
+    }
+
+    const filtered = advertisers?.filter(advertiser => advertiser.name.toLocaleLowerCase().includes(search.toLocaleLowerCase())) ?? []
+    return (
+        <div className="flex flex-1 flex-col gap-6 p-4 md:p-6">
+            <div><h1 className="text-2xl font-bold">Anunciantes</h1><p className="text-muted-foreground">Gestiona los anunciantes de tus campañas de publicidad.</p></div>
+            {actionError && <p role="alert" className="text-destructive">{actionError}</p>}
+            <form onSubmit={save} className="flex flex-wrap items-end gap-3 rounded-md border p-4">
+                <div className="grid gap-2 flex-1 min-w-48">
+                    <Label htmlFor="advertiser-name">{editingId ? "Editar anunciante" : "Nuevo anunciante"}</Label>
+                    <Input id="advertiser-name" value={name} onChange={event => setName(event.target.value)} required maxLength={100} disabled={busy || !!error || isLoading} placeholder="Nombre comercial" />
+                </div>
+                <Button type="submit" disabled={busy || !name.trim() || !!error || isLoading}>{busy ? "Guardando..." : editingId ? "Guardar cambios" : "Crear anunciante"}</Button>
+                {editingId && <Button type="button" variant="outline" disabled={busy} onClick={resetForm}>Cancelar edición</Button>}
+            </form>
+            <Input aria-label="Buscar anunciantes" placeholder="Buscar anunciantes..." value={search} onChange={event => setSearch(event.target.value)} />
+            {error ? <div role="alert">No se pudieron cargar los anunciantes. <Button variant="outline" onClick={() => mutate()}>Reintentar</Button></div> :
+                <div className="rounded-md border"><Table>
+                    <TableHeader><TableRow><TableHead>Nombre</TableHead><TableHead className="text-right">Acciones</TableHead></TableRow></TableHeader>
+                    <TableBody>
+                        {isLoading ? <TableRow><TableCell colSpan={2}>Cargando...</TableCell></TableRow> : filtered.length === 0 ? <TableRow><TableCell colSpan={2}>{search ? "No hay anunciantes que coincidan." : "Todavía no hay anunciantes. Crea el primero para asignarle campañas."}</TableCell></TableRow> : filtered.map(advertiser =>
+                            <TableRow key={advertiser.id}>
+                                <TableCell>{advertiser.name}</TableCell>
+                                <TableCell className="text-right">
+                                    <Button variant="ghost" size="icon" aria-label={`Editar ${advertiser.name}`} disabled={busy} onClick={() => { setEditingId(advertiser.id); setName(advertiser.name); setActionError(null); document.getElementById("advertiser-name")?.focus() }}><Pencil className="h-4 w-4" /></Button>
+                                    <Button variant="ghost" size="icon" aria-label={`Eliminar ${advertiser.name}`} disabled={busy} onClick={() => { setDeleteTarget(advertiser); setActionError(null) }}><Trash2 className="h-4 w-4 text-destructive" /></Button>
+                                </TableCell>
+                            </TableRow>
+                        )}
+                    </TableBody>
+                </Table></div>
+            }
+            <AlertDialog open={!!deleteTarget} onOpenChange={open => { if (!open && !busy) setDeleteTarget(null) }}>
+                <AlertDialogContent>
+                    <AlertDialogHeader><AlertDialogTitle>¿Eliminar {deleteTarget?.name}?</AlertDialogTitle><AlertDialogDescription>Si tiene campañas activas, no se eliminará nada. En caso contrario, se eliminarán también sus campañas inactivas, banners y métricas asociados. Esta acción es irreversible.</AlertDialogDescription></AlertDialogHeader>
+                    <AlertDialogFooter><AlertDialogCancel disabled={busy}>Cancelar</AlertDialogCancel><Button variant="destructive" disabled={busy} onClick={remove}>{busy ? "Eliminando..." : "Eliminar anunciante"}</Button></AlertDialogFooter>
+                </AlertDialogContent>
+            </AlertDialog>
+        </div>
+    )
+}
diff --git a/frontend/src/components/app-sidebar.tsx b/frontend/src/components/app-sidebar.tsx
index 10a00fb..fe46419 100644
--- a/frontend/src/components/app-sidebar.tsx
+++ b/frontend/src/components/app-sidebar.tsx
@@ -96,6 +96,11 @@ const data = {
     }
   ],
   navAdvertising: [
+    {
+      title: "Anunciantes",
+      url: "/dashboard/ads/advertisers",
+      icon: IconUsers,
+    },
     {
       title: "Campañas",
       url: "/dashboard/ads/campaigns",
```

### H0.2 — `26b9471`: fix(ads): unwrap edit page params for Next 16

```diff
diff --git a/frontend/src/app/dashboard/ads/banners/[id]/page.tsx b/frontend/src/app/dashboard/ads/banners/[id]/page.tsx
index 8bf1344..e765a1a 100644
--- a/frontend/src/app/dashboard/ads/banners/[id]/page.tsx
+++ b/frontend/src/app/dashboard/ads/banners/[id]/page.tsx
@@ -1,14 +1,16 @@
 "use client"
 
+import { use } from "react"
 import { BannerForm } from "@/components/ads/banner-form"
 import useSWR from "swr"
 import { Loader2 } from "lucide-react"
 
 import { fetcher } from "@/lib/fetcher"
 
-export default function EditBannerPage({ params }: { params: { id: string } }) {
+export default function EditBannerPage({ params }: { params: Promise<{ id: string }> }) {
+    const { id } = use(params)
     const { data: banner, error, isLoading } = useSWR(
-        `/api/v1/ads/banners/${params.id}`,
+        `/api/v1/ads/banners/${id}`,
         fetcher
     )
 
diff --git a/frontend/src/app/dashboard/ads/campaigns/[id]/page.tsx b/frontend/src/app/dashboard/ads/campaigns/[id]/page.tsx
index f8d352f..cb3a519 100644
--- a/frontend/src/app/dashboard/ads/campaigns/[id]/page.tsx
+++ b/frontend/src/app/dashboard/ads/campaigns/[id]/page.tsx
@@ -1,14 +1,16 @@
 "use client"
 
+import { use } from "react"
 import { CampaignForm } from "@/components/ads/campaign-form"
 import useSWR from "swr"
 import { Loader2 } from "lucide-react"
 
 import { fetcher } from "@/lib/fetcher"
 
-export default function EditCampaignPage({ params }: { params: { id: string } }) {
+export default function EditCampaignPage({ params }: { params: Promise<{ id: string }> }) {
+    const { id } = use(params)
     const { data: campaign, error, isLoading } = useSWR(
-        `/api/v1/ads/campaigns/${params.id}`,
+        `/api/v1/ads/campaigns/${id}`,
         fetcher
     )
```

### H0.3 — `3e7461f`: feat(ads): add shared CSRF requests and form validation

```diff
diff --git a/backend/ads/api.py b/backend/ads/api.py
index 64f1eec..6f69201 100644
--- a/backend/ads/api.py
+++ b/backend/ads/api.py
@@ -1,8 +1,10 @@
 from django.shortcuts import get_object_or_404
 from ninja import Router
+from ninja.errors import HttpError
+from django.utils import timezone
 from typing import List
 from uuid import UUID
-from core.models import Brand
+from core.models import Brand, MediaAsset
 from publishing.models import Site
 from .models import AdCampaign, AdBanner
 from .schemas import (
@@ -12,6 +14,28 @@ from .schemas import (
 
 router = Router(tags=["Advertising"])
 
+
+def validate_campaign_dates(start, end):
+    if start is None or end is None:
+        raise HttpError(422, "Las fechas de inicio y fin son obligatorias.")
+    if timezone.is_naive(start):
+        start = timezone.make_aware(start)
+    if timezone.is_naive(end):
+        end = timezone.make_aware(end)
+    if start > end:
+        raise HttpError(422, "La fecha de inicio debe ser anterior o igual a la fecha de fin.")
+
+
+def validate_banner(active, image_id, script_code):
+    if script_code:
+        raise HttpError(422, "Los scripts publicitarios están deshabilitados.")
+    if active and not image_id:
+        raise HttpError(422, "Un banner activo requiere una imagen.")
+    if image_id:
+        image = get_object_or_404(MediaAsset, id=image_id)
+        if active and not image.file:
+            raise HttpError(422, "Un banner activo requiere una imagen.")
+
 # --- Campaign Endpoints ---
 
 @router.get("/campaigns", response=List[AdCampaignOut])
@@ -20,6 +44,7 @@ def list_campaigns(request):
 
 @router.post("/campaigns", response=AdCampaignOut)
 def create_campaign(request, payload: AdCampaignCreate):
+    validate_campaign_dates(payload.start_date, payload.end_date)
     brand = get_object_or_404(Brand, id=payload.brand_id)
     campaign = AdCampaign.objects.create(
         brand=brand,
@@ -38,8 +63,10 @@ def get_campaign(request, campaign_id: UUID):
 @router.put("/campaigns/{campaign_id}", response=AdCampaignOut)
 def update_campaign(request, campaign_id: UUID, payload: AdCampaignUpdate):
     campaign = get_object_or_404(AdCampaign, id=campaign_id)
-    
-    for attr, value in payload.dict(exclude_unset=True).items():
+    data = payload.dict(exclude_unset=True)
+    validate_campaign_dates(data.get("start_date", campaign.start_date), data.get("end_date", campaign.end_date))
+
+    for attr, value in data.items():
         if attr == 'brand_id' and value:
              brand = get_object_or_404(Brand, id=value)
              campaign.brand = brand
@@ -66,6 +93,7 @@ def list_banners(request, campaign_id: UUID = None):
 
 @router.post("/banners", response=AdBannerOut)
 def create_banner(request, payload: AdBannerCreate):
+    validate_banner(payload.is_active, payload.image_id, payload.script_code)
     campaign = get_object_or_404(AdCampaign, id=payload.campaign_id)
     
     # Handle M2M separately
@@ -100,6 +128,7 @@ def update_banner(request, banner_id: UUID, payload: AdBannerUpdate):
     banner = get_object_or_404(AdBanner, id=banner_id)
     
     data = payload.dict(exclude_unset=True)
+    validate_banner(data.get("is_active", banner.is_active), data.get("image_id", banner.image_id), data.get("script_code", banner.script_code))
     target_site_ids = data.pop('target_site_ids', None)
     
     for attr, value in data.items():
diff --git a/frontend/src/app/dashboard/ads/advertisers/page.tsx b/frontend/src/app/dashboard/ads/advertisers/page.tsx
index 9b64339..886f0c8 100644
--- a/frontend/src/app/dashboard/ads/advertisers/page.tsx
+++ b/frontend/src/app/dashboard/ads/advertisers/page.tsx
@@ -4,6 +4,7 @@ import { useState, type FormEvent } from "react"
 import useSWR from "swr"
 import { Pencil, Trash2 } from "lucide-react"
 import { toast } from "sonner"
+import { apiFetch } from "@/lib/api-fetch"
 import { fetcher } from "@/lib/fetcher"
 import { Button } from "@/components/ui/button"
 import { Input } from "@/components/ui/input"
@@ -14,24 +15,6 @@ import { AlertDialog, AlertDialogCancel, AlertDialogContent, AlertDialogDescript
 interface Advertiser { id: string; name: string }
 const API_URL = "/api/v1/core/brands"
 
-async function advertiserMutation(url: string, method: string, name?: string) {
-    // Refresh the CSRF cookie for older sessions as well as newly logged-in users.
-    const auth = await fetch("/api/v1/core/check-auth", { credentials: "include" })
-    if (!auth.ok) throw new Error("La sesión ha caducado. Inicia sesión de nuevo.")
-    const token = document.cookie.split("; ").find(cookie => cookie.startsWith("csrftoken="))?.slice("csrftoken=".length)
-    if (!token) throw new Error("No se pudo obtener el token de seguridad. Recarga la página.")
-    const response = await fetch(url, {
-        method,
-        credentials: "include",
-        headers: { "Content-Type": "application/json", "X-CSRFToken": decodeURIComponent(token) },
-        ...(name !== undefined ? { body: JSON.stringify({ name }) } : {}),
-    })
-    if (!response.ok) {
-        const body = await response.json().catch(() => null)
-        throw new Error(typeof body?.detail === "string" ? body.detail : "No se pudo guardar el cambio. Comprueba el nombre e inténtalo de nuevo.")
-    }
-}
-
 export default function AdvertisersPage() {
     const { data: advertisers, error, isLoading, mutate } = useSWR<Advertiser[]>(API_URL, fetcher)
     const [name, setName] = useState("")
@@ -49,7 +32,7 @@ export default function AdvertisersPage() {
         setBusy(true)
         setActionError(null)
         try {
-            await advertiserMutation(editingId ? `${API_URL}/${editingId}` : API_URL, editingId ? "PUT" : "POST", name.trim())
+            await apiFetch(editingId ? `${API_URL}/${editingId}` : API_URL, { method: editingId ? "PUT" : "POST", body: JSON.stringify({ name: name.trim() }) })
             toast.success(editingId ? "Anunciante actualizado" : "Anunciante creado")
             resetForm()
             await mutate()
@@ -63,7 +46,7 @@ export default function AdvertisersPage() {
         setBusy(true)
         setActionError(null)
         try {
-            await advertiserMutation(`${API_URL}/${deleteTarget.id}`, "DELETE")
+            await apiFetch(`${API_URL}/${deleteTarget.id}`, { method: "DELETE" })
             if (editingId === deleteTarget.id) resetForm()
             setDeleteTarget(null)
             toast.success("Anunciante eliminado")
diff --git a/frontend/src/app/dashboard/ads/banners/page.tsx b/frontend/src/app/dashboard/ads/banners/page.tsx
index 164355d..2198c9a 100644
--- a/frontend/src/app/dashboard/ads/banners/page.tsx
+++ b/frontend/src/app/dashboard/ads/banners/page.tsx
@@ -1,5 +1,7 @@
 "use client"
 
+import { apiFetch } from "@/lib/api-fetch"
+
 import { useState } from "react"
 import useSWR from "swr"
 import Link from "next/link"
@@ -65,13 +67,12 @@ export default function BannersPage() {
     const onConfirmDelete = async () => {
         if (!deleteId) return
         try {
-            const res = await fetch(`${API_URL}/${deleteId}`, { method: "DELETE" })
-            if (!res.ok) throw new Error("Error al eliminar")
+            await apiFetch(`${API_URL}/${deleteId}`, { method: "DELETE" })
             toast.success("Banner eliminado")
             mutateBanners()
         } catch (err) {
             console.error(err)
-            toast.error("No se pudo eliminar el banner")
+            toast.error(err instanceof Error ? err.message : "No se pudo eliminar el banner")
         } finally {
             setDeleteId(null)
         }
diff --git a/frontend/src/app/dashboard/ads/campaigns/page.tsx b/frontend/src/app/dashboard/ads/campaigns/page.tsx
index 6512537..6a63f71 100644
--- a/frontend/src/app/dashboard/ads/campaigns/page.tsx
+++ b/frontend/src/app/dashboard/ads/campaigns/page.tsx
@@ -1,5 +1,7 @@
 "use client"
 
+import { apiFetch } from "@/lib/api-fetch"
+
 import { useState } from "react"
 import useSWR, { mutate } from "swr"
 import Link from "next/link"
@@ -64,13 +66,12 @@ export default function CampaignsPage() {
     const onConfirmDelete = async () => {
         if (!deleteId) return
         try {
-            const res = await fetch(`${API_URL}/${deleteId}`, { method: "DELETE" })
-            if (!res.ok) throw new Error("Error al eliminar")
+            await apiFetch(`${API_URL}/${deleteId}`, { method: "DELETE" })
             toast.success("Campaña eliminada")
             mutateCampaigns()
         } catch (err) {
             console.error(err)
-            toast.error("No se pudo eliminar la campaña")
+            toast.error(err instanceof Error ? err.message : "No se pudo eliminar la campaña")
         } finally {
             setDeleteId(null)
         }
diff --git a/frontend/src/components/ads/banner-form.tsx b/frontend/src/components/ads/banner-form.tsx
index 5127b81..2a0d5ad 100644
--- a/frontend/src/components/ads/banner-form.tsx
+++ b/frontend/src/components/ads/banner-form.tsx
@@ -31,12 +31,11 @@ import {
 import { Switch } from "@/components/ui/switch"
 import { MultiSelect } from "@/components/ui/multi-select"
 
-const fetcher = (url: string) => fetch(url).then((res) => res.json())
+import { apiFetch } from "@/lib/api-fetch"
+import { fetcher } from "@/lib/fetcher"
 
 const formSchema = z.object({
-    campaign_id: z.string({
-        required_error: "Selecciona una campaña.",
-    }),
+    campaign_id: z.string().uuid("Selecciona una campaña."),
     name: z.string().min(2, "El nombre es requerido."),
     width: z.coerce.number().min(1, "Ancho requerido."),
     height: z.coerce.number().min(1, "Alto requerido."),
@@ -48,6 +47,9 @@ const formSchema = z.object({
     is_global: z.boolean().default(false),
     target_site_ids: z.array(z.string()).optional(),
     is_active: z.boolean().default(true),
+}).refine(values => !values.is_active || !!values.image_id, {
+    message: "Un banner activo requiere una imagen.",
+    path: ["image_id"],
 })
 
 interface BannerFormProps {
@@ -72,7 +74,7 @@ export function BannerForm({ initialData }: BannerFormProps) {
             size_label: initialData?.size_label || "",
             position: initialData?.position || "sidebar",
             link: initialData?.link || "",
-            script_code: initialData?.script_code || "",
+            script_code: "",
             image_id: initialData?.image_id || null,
             is_global: initialData?.is_global !== undefined ? initialData.is_global : false,
             target_site_ids: initialData?.target_sites ? initialData.target_sites.map((s: any) => s.id || s.domain) : [], // check format
@@ -105,27 +107,24 @@ export function BannerForm({ initialData }: BannerFormProps) {
         formData.append("file", file);
 
         try {
-            const res = await fetch("/api/v1/core/media", {
+            const data = await apiFetch<{ id: string; file: string }>("/api/v1/core/media", {
                 method: "POST",
                 body: formData,
             });
 
-            if (!res.ok) throw new Error("Error al subir imagen");
-
-            const data = await res.json();
-            form.setValue("image_id", data.id, { shouldDirty: true });
+            form.setValue("image_id", data.id, { shouldDirty: true, shouldValidate: true });
             setPreviewImage(data.file); // Assuming API returns 'file' as URL
             toast.success("Imagen subida correctamente");
-        } catch (err: any) {
+        } catch (err: unknown) {
             console.error(err);
-            toast.error("Error al subir la imagen");
+            toast.error(err instanceof Error ? err.message : "Error al subir la imagen");
         } finally {
             setUploading(false);
         }
     };
 
     const removeImage = () => {
-        form.setValue("image_id", null, { shouldDirty: true });
+        form.setValue("image_id", null, { shouldDirty: true, shouldValidate: true });
         setPreviewImage(null);
     };
 
@@ -140,12 +139,12 @@ export function BannerForm({ initialData }: BannerFormProps) {
             const method = initialData ? "PUT" : "POST"
 
             // Clean up
-            const payload = { ...values }
+            const payload = { ...values, script_code: "" }
             if (payload.is_global) {
                 payload.target_site_ids = []
             }
 
-            const res = await fetch(url, {
+            await apiFetch(url, {
                 method,
                 headers: {
                     "Content-Type": "application/json",
@@ -153,16 +152,11 @@ export function BannerForm({ initialData }: BannerFormProps) {
                 body: JSON.stringify(payload),
             })
 
-            if (!res.ok) {
-                const errorData = await res.json()
-                throw new Error(errorData.detail || "Error al guardar banner")
-            }
-
             toast.success(initialData ? "Banner actualizado" : "Banner creado")
             router.push("/dashboard/ads/banners")
             router.refresh()
-        } catch (error: any) {
-            toast.error(error.message)
+        } catch (error: unknown) {
+            toast.error(error instanceof Error ? error.message : "Error al guardar banner")
         } finally {
             setIsSubmitting(false)
         }
@@ -333,6 +327,7 @@ export function BannerForm({ initialData }: BannerFormProps) {
                     <div className="space-y-6">
                         <div className="rounded-md border p-4 space-y-4">
                             <FormLabel>Creatividad (Imagen)</FormLabel>
+                            <FormField control={form.control} name="image_id" render={() => <FormItem><FormMessage /></FormItem>} />
 
                             {previewImage ? (
                                 <div className="space-y-2">
@@ -400,11 +395,12 @@ export function BannerForm({ initialData }: BannerFormProps) {
                                 render={({ field }) => (
                                     <FormItem>
                                         <FormDescription>
-                                            Si proporcionas un script (ej. Google Ads), se usará en lugar de la imagen.
+                                            Los scripts están deshabilitados. Usa una imagen para la creatividad.
                                         </FormDescription>
                                         <FormControl>
                                             <Textarea
-                                                placeholder="<script>...</script>"
+                                                disabled
+                                                placeholder="Scripts deshabilitados"
                                                 className="font-mono text-xs min-h-[150px]"
                                                 {...field}
                                             />
@@ -418,7 +414,7 @@ export function BannerForm({ initialData }: BannerFormProps) {
                 </div>
 
                 <div className="flex gap-4">
-                    <Button type="submit" disabled={isSubmitting}>
+                    <Button type="submit" disabled={isSubmitting || uploading}>
                         {isSubmitting && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
                         {initialData ? "Actualizar Banner" : "Crear Banner"}
                     </Button>
diff --git a/frontend/src/components/ads/campaign-form.tsx b/frontend/src/components/ads/campaign-form.tsx
index c0460a8..eb7747f 100644
--- a/frontend/src/components/ads/campaign-form.tsx
+++ b/frontend/src/components/ads/campaign-form.tsx
@@ -1,6 +1,6 @@
 "use client"
 
-import { useEffect, useState } from "react"
+import { useState } from "react"
 import { useRouter } from "next/navigation"
 import { zodResolver } from "@hookform/resolvers/zod"
 import { useForm } from "react-hook-form"
@@ -33,15 +33,14 @@ import { Switch } from "@/components/ui/switch"
 import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover"
 import { Calendar } from "@/components/ui/calendar"
 
-const fetcher = (url: string) => fetch(url).then((res) => res.json())
+import { apiFetch } from "@/lib/api-fetch"
+import { fetcher } from "@/lib/fetcher"
 
 const formSchema = z.object({
     name: z.string().min(2, {
         message: "El nombre debe tener al menos 2 caracteres.",
     }),
-    brand_id: z.string({
-        required_error: "Selecciona una marca/cliente.",
-    }),
+    brand_id: z.string().uuid("Selecciona un anunciante."),
     start_date: z.date({
         required_error: "Fecha de inicio requerida.",
     }),
@@ -52,6 +51,9 @@ const formSchema = z.object({
         message: "El presupuesto debe ser mayor o igual a 0.",
     }),
     is_active: z.boolean().default(true),
+}).refine(values => values.start_date <= values.end_date, {
+    message: "La fecha de inicio debe ser anterior o igual a la fecha de fin.",
+    path: ["end_date"],
 })
 
 interface CampaignFormProps {
@@ -62,33 +64,8 @@ export function CampaignForm({ initialData }: CampaignFormProps) {
     const router = useRouter()
     const [isSubmitting, setIsSubmitting] = useState(false)
 
-    // Fetch Brands (Organization -> Brand ?)
-    // Wait, Brand is a child of Organization. 
-    // We should probably list all Brands if we are SuperAdmin, or filter.
-    // Assuming /api/v1/core/brands exists? Or we check core/api/views.py
-    // Let's assume we can fetch brands. If not, I'll need to create that endpoint or use organisations.
-    // Plan said "Advertising Campaign owned by a Brand".
-    // I need to check if there is an endpoint for brands.
-    // If not, I might have to add it or use organisations temporarily.
-    // Let's optimisticly assume /api/core/brands exists or similar.
-    // Wait, in previous steps I saw `core/models.py` has `Brand`.
-    // I did not see `Brand` in `backend/core/api/views.py` specifically?
-    // UseSWR will fail if 404.
-
-    // For now, I will use a mock or try to fetch.
-    // Actually, `backend/core/api/views.py` was "OTHER OPEN DOCUMENTS".
-    // I should check it quickly in a parallel tool or just assume I might need to add it.
-    // I'll add `api/core/brands` support if needed.
-    // Let's assume there is none and I'll use Organizations for now if I can't find Brands?
-    // No, I must use Brands as per model.
-    // I will try to fetch `/api/core/brands` if it exists.
-
-    // Quick fix: I'll include a way to fetch brands in the same file or assume it works.
-    // If I didn't add the endpoint, I must add it.
-    // I did NOT add `Brand` endpoint in my plan explicitly (oops).
-    // I'll check `backend/core/api/views.py` AFTER this write or rely on a "Brand" select that might be empty.
-
-    const { data: brands } = useSWR<any[]>("/api/v1/core/brands", fetcher)
+    const { data: brands, error: brandsError, isLoading: brandsLoading } = useSWR<{ id: string; name: string }[]>("/api/v1/core/brands", fetcher)
+    const canSave = !brandsLoading && !brandsError && !!brands?.length
 
     const form = useForm<z.infer<typeof formSchema>>({
         resolver: zodResolver(formSchema),
@@ -103,6 +80,7 @@ export function CampaignForm({ initialData }: CampaignFormProps) {
     })
 
     async function onSubmit(values: z.infer<typeof formSchema>) {
+        if (!canSave) return
         setIsSubmitting(true)
         try {
             const url = initialData
@@ -110,7 +88,7 @@ export function CampaignForm({ initialData }: CampaignFormProps) {
                 : "/api/v1/ads/campaigns"
             const method = initialData ? "PUT" : "POST"
 
-            const res = await fetch(url, {
+            await apiFetch(url, {
                 method,
                 headers: {
                     "Content-Type": "application/json",
@@ -122,16 +100,11 @@ export function CampaignForm({ initialData }: CampaignFormProps) {
                 }),
             })
 
-            if (!res.ok) {
-                const errorData = await res.json()
-                throw new Error(errorData.detail || "Error al guardar campaña")
-            }
-
             toast.success(initialData ? "Campaña actualizada" : "Campaña creada")
             router.push("/dashboard/ads/campaigns")
             router.refresh()
-        } catch (error: any) {
-            toast.error(error.message)
+        } catch (error: unknown) {
+            toast.error(error instanceof Error ? error.message : "Error al guardar campaña")
         } finally {
             setIsSubmitting(false)
         }
@@ -303,8 +276,9 @@ export function CampaignForm({ initialData }: CampaignFormProps) {
                     )}
                 />
 
+                {brandsError ? <p role="alert">No se pudieron cargar los anunciantes.</p> : !brandsLoading && !brands?.length && <p role="alert">Crea un anunciante en Anunciantes antes de guardar una campaña.</p>}
                 <div className="flex gap-4">
-                    <Button type="submit" disabled={isSubmitting}>
+                    <Button type="submit" disabled={isSubmitting || !canSave}>
                         {isSubmitting && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
                         {initialData ? "Actualizar Campaña" : "Crear Campaña"}
                     </Button>
diff --git a/frontend/src/lib/api-fetch.ts b/frontend/src/lib/api-fetch.ts
new file mode 100644
index 0000000..38e05f3
--- /dev/null
+++ b/frontend/src/lib/api-fetch.ts
@@ -0,0 +1,48 @@
+export class ApiError extends Error {
+    constructor(message: string, public status: number, public info: unknown) {
+        super(message)
+        this.name = "ApiError"
+    }
+}
+
+function errorMessage(body: unknown, status: number): string {
+    if (body && typeof body === "object" && "detail" in body) {
+        const detail = body.detail
+        if (typeof detail === "string") return detail
+        if (Array.isArray(detail)) {
+            return detail.map(item => {
+                const field = Array.isArray(item.loc) ? item.loc.slice(1).join(".") : ""
+                return `${field ? `${field}: ` : ""}${item.msg || "Valor inválido"}`
+            }).join("; ")
+        }
+    }
+    return `No se pudo completar la solicitud (${status}).`
+}
+
+async function checkedFetch(url: string, options: RequestInit): Promise<Response> {
+    const response = await fetch(url, { ...options, credentials: "include" })
+    if (!response.ok) {
+        const body: unknown = await response.json().catch(() => null)
+        throw new ApiError(errorMessage(body, response.status), response.status, body)
+    }
+    return response
+}
+
+export async function apiFetch<T = unknown>(url: string, options: RequestInit = {}): Promise<T> {
+    const headers = new Headers(options.headers)
+    const method = (options.method || "GET").toUpperCase()
+    if (!["GET", "HEAD", "OPTIONS"].includes(method)) {
+        await checkedFetch("/api/v1/core/check-auth", { method: "GET" })
+        const token = document.cookie.split(";").map(cookie => cookie.trim())
+            .find(cookie => cookie.startsWith("csrftoken="))?.slice("csrftoken=".length)
+        if (!token) throw new Error("No se pudo obtener el token de seguridad. Recarga la página.")
+        headers.set("X-CSRFToken", decodeURIComponent(token))
+    }
+    // Let the browser supply the multipart boundary for file uploads.
+    if (typeof options.body === "string" && !headers.has("Content-Type")) {
+        headers.set("Content-Type", "application/json")
+    }
+    const response = await checkedFetch(url, { ...options, headers })
+    if (response.status === 204) return undefined as T
+    return response.json() as Promise<T>
+}
```

### H0.4 — `4ce760d`: feat(ads): link empty campaign form to advertiser creation

```diff
diff --git a/frontend/src/components/ads/campaign-form.tsx b/frontend/src/components/ads/campaign-form.tsx
index eb7747f..2cee201 100644
--- a/frontend/src/components/ads/campaign-form.tsx
+++ b/frontend/src/components/ads/campaign-form.tsx
@@ -1,6 +1,7 @@
 "use client"
 
 import { useState } from "react"
+import Link from "next/link"
 import { useRouter } from "next/navigation"
 import { zodResolver } from "@hookform/resolvers/zod"
 import { useForm } from "react-hook-form"
@@ -276,7 +277,16 @@ export function CampaignForm({ initialData }: CampaignFormProps) {
                     )}
                 />
 
-                {brandsError ? <p role="alert">No se pudieron cargar los anunciantes.</p> : !brandsLoading && !brands?.length && <p role="alert">Crea un anunciante en Anunciantes antes de guardar una campaña.</p>}
+                {brandsError ? (
+                    <p role="alert">No se pudieron cargar los anunciantes.</p>
+                ) : !brandsLoading && !brands?.length && (
+                    <div role="alert" className="space-y-2">
+                        <p>Crea un anunciante antes de guardar una campaña.</p>
+                        <Button asChild variant="outline">
+                            <Link href="/dashboard/ads/advertisers">Crear anunciante</Link>
+                        </Button>
+                    </div>
+                )}
                 <div className="flex gap-4">
                     <Button type="submit" disabled={isSubmitting || !canSave}>
                         {isSubmitting && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
```

### H0.5 — `67d2430`: test(ads): replace anonymous tests with authenticated contracts

```diff
diff --git a/backend/ads/tests.py b/backend/ads/tests.py
index 40a1212..44b62e3 100644
--- a/backend/ads/tests.py
+++ b/backend/ads/tests.py
@@ -1,86 +1,274 @@
-from django.test import TestCase, Client
-from django.utils import timezone
-from core.models import Brand, Organization
-from ads.models import AdCampaign, AdBanner
+"""Authenticated contracts for the advertising backoffice API."""
 import json
 from datetime import timedelta
+from uuid import uuid4
+
+from django.contrib.auth.models import User
+from django.core.files.uploadedfile import SimpleUploadedFile
+from django.test import Client, TestCase, override_settings
+from django.utils import timezone
+
+from ads.models import AdBanner, AdCampaign
+from core.models import Brand, MediaAsset, Organization, UserProfile
+from publishing.models import Site
 
-class AdsApiTest(TestCase):
+
+@override_settings(STORAGES={
+    "default": {"BACKEND": "django.core.files.storage.InMemoryStorage"},
+    "staticfiles": {"BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage"},
+})
+class AdsApiContractTests(TestCase):
     def setUp(self):
-        self.client = Client()
         self.org = Organization.objects.create(
-            name="Test Org", 
-            tax_id="B123", 
-            billing_email="test@test.com"
+            name="Sales", tax_id="TEST-SALES", billing_email="sales@example.com"
         )
-        self.brand = Brand.objects.create(
-            organization=self.org, 
-            name="Test Brand"
+        self.user = User.objects.create_user(username="sales", password="test-password")
+        UserProfile.objects.create(user=self.user, organization=self.org)
+        self.brand = Brand.objects.create(name="Advertiser", organization=self.org)
+        self.site = Site.objects.create(name="Portal", domain="portal.example.com")
+        self.start = timezone.now().replace(microsecond=0)
+        self.end = self.start + timedelta(days=30)
+        self.image = MediaAsset.objects.create(file="contract-banner.gif")
+        self.client = Client(enforce_csrf_checks=True)
+        self.client.force_login(self.user)
+        response = self.client.get("/api/v1/core/check-auth")
+        self.assertEqual(response.status_code, 200)
+        self.csrf = self.client.cookies["csrftoken"].value
+
+    def write(self, method, path, data=None, csrf=True, client=None):
+        return getattr(client or self.client, method)(
+            "/api/v1/" + path,
+            data=json.dumps(data if data is not None else {}),
+            content_type="application/json",
+            **({"HTTP_X_CSRFTOKEN": self.csrf} if csrf else {}),
         )
-        
-    def test_create_campaign(self):
-        start = timezone.now()
-        end = start + timedelta(days=30)
-        
-        payload = {
-            "brand_id": str(self.brand.id),
-            "name": "Summer Campaign",
-            "start_date": start.isoformat(),
-            "end_date": end.isoformat(),
-            "budget": 1000.0,
-            "is_active": True
+
+    def campaign_data(self, **changes):
+        return {
+            "brand_id": str(self.brand.id), "name": "Campaign",
+            "start_date": self.start.isoformat(), "end_date": self.end.isoformat(),
+            "budget": 1000, "is_active": True, **changes,
         }
-        
-        response = self.client.post(
-            "/api/v1/ads/campaigns", 
-            data=json.dumps(payload), 
-            content_type="application/json"
+
+    def campaign(self):
+        return AdCampaign.objects.create(
+            brand=self.brand, name="Campaign", start_date=self.start, end_date=self.end
         )
-        
+
+    def banner_data(self, campaign, **changes):
+        return {
+            "campaign_id": str(campaign.id), "name": "Banner",
+            "image_id": str(self.image.id), "width": 300, "height": 250,
+            "position": "sidebar", "link": "https://example.com/offer",
+            "target_site_ids": [str(self.site.id)], "is_active": True, **changes,
+        }
+
+    def test_campaign_crud(self):
+        response = self.write("post", "ads/campaigns", self.campaign_data())
         self.assertEqual(response.status_code, 200)
         data = response.json()
-        self.assertEqual(data["name"], "Summer Campaign")
+        path = f"ads/campaigns/{data['id']}"
+        self.assertEqual(data["brand_id"], str(self.brand.id))
+        self.assertEqual(data["brand_name"], self.brand.name)
+        response = self.client.get("/api/v1/ads/campaigns")
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual([row["id"] for row in response.json()], [data["id"]])
+        response = self.client.get("/api/v1/" + path)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.json()["name"], "Campaign")
+        response = self.write("put", path, {"name": "Edited", "budget": 1250})
+        self.assertEqual(response.status_code, 200)
+        campaign = AdCampaign.objects.get(id=data["id"])
+        self.assertEqual(campaign.name, "Edited")
+        self.assertEqual(campaign.budget, 1250)
+        self.assertEqual(campaign.start_date, self.start)
+        response = self.write("delete", path)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.json(), {"success": True})
+        self.assertFalse(AdCampaign.objects.filter(id=data["id"]).exists())
+
+    def test_banner_crud_and_campaign_filter(self):
+        campaign = self.campaign()
+        response = self.write("post", "ads/banners", self.banner_data(campaign))
+        self.assertEqual(response.status_code, 200)
+        data = response.json()
+        path = f"ads/banners/{data['id']}"
+        self.assertEqual(data["image_id"], str(self.image.id))
+        self.assertTrue(data["image_url"])
+        self.assertEqual(data["campaign_id"], str(campaign.id))
+        self.assertEqual(data["target_sites"], [{"id": str(self.site.id), "domain": self.site.domain}])
+        other = self.campaign()
+        AdBanner.objects.create(campaign=other, is_active=False)
+        response = self.client.get(f"/api/v1/ads/banners?campaign_id={campaign.id}")
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual([row["id"] for row in response.json()], [data["id"]])
+        response = self.write("put", path, {"name": "Edited", "target_site_ids": []})
+        self.assertEqual(response.status_code, 200)
+        response = self.client.get("/api/v1/" + path)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.json()["name"], "Edited")
+        self.assertEqual(response.json()["target_sites"], [])
+        response = self.write("delete", path)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.json(), {"success": True})
+        self.assertFalse(AdBanner.objects.filter(id=data["id"]).exists())
+
+    def test_anonymous_requests_rejected(self):
+        campaign = self.campaign()
+        banner = AdBanner.objects.create(campaign=campaign, is_active=False)
+        anonymous = Client()
+        for path in ["ads/campaigns", f"ads/campaigns/{campaign.id}", "ads/banners", f"ads/banners/{banner.id}"]:
+            with self.subTest(path=path):
+                self.assertEqual(anonymous.get("/api/v1/" + path).status_code, 401)
+        for method, path, data in self.mutations(campaign, banner):
+            with self.subTest(method=method, path=path):
+                self.assertEqual(self.write(method, path, data, csrf=False, client=anonymous).status_code, 401)
         self.assertEqual(AdCampaign.objects.count(), 1)
-        
-    def test_create_banner_and_list(self):
-        # Create Campaign first
-        campaign = AdCampaign.objects.create(
-            brand=self.brand,
-            name="Test Campaign",
-            start_date=timezone.now(),
-            end_date=timezone.now() + timedelta(days=30)
-        )
-        
-        # Create Banner via API
-        payload = {
-            "campaign_id": str(campaign.id),
-            "name": "Banner 1",
-            "width": 300,
-            "height": 250,
-            "position": "sidebar",
-            "is_global": True,
-            "is_active": True
-        }
-        
+        self.assertEqual(AdBanner.objects.count(), 1)
+        campaign.refresh_from_db()
+        self.assertEqual(campaign.name, "Campaign")
+
+    def mutations(self, campaign, banner):
+        return [
+            ("post", "ads/campaigns", self.campaign_data()),
+            ("put", f"ads/campaigns/{campaign.id}", {"name": "Changed"}),
+            ("delete", f"ads/campaigns/{campaign.id}", {}),
+            ("post", "ads/banners", self.banner_data(campaign)),
+            ("put", f"ads/banners/{banner.id}", {"name": "Changed"}),
+            ("delete", f"ads/banners/{banner.id}", {}),
+        ]
+
+    def test_mutations_require_csrf(self):
+        campaign = self.campaign()
+        banner = AdBanner.objects.create(campaign=campaign, is_active=False)
+        for method, path, data in self.mutations(campaign, banner):
+            with self.subTest(method=method, path=path):
+                self.assertEqual(self.write(method, path, data, csrf=False).status_code, 403)
+        campaign.refresh_from_db()
+        banner.refresh_from_db()
+        self.assertEqual(campaign.name, "Campaign")
+        self.assertEqual(banner.name, "Banner")
+        self.assertEqual(AdCampaign.objects.count(), 1)
+        self.assertEqual(AdBanner.objects.count(), 1)
+
+    def test_invalid_csrf_rejected(self):
         response = self.client.post(
-            "/api/v1/ads/banners",
-            data=json.dumps(payload),
-            content_type="application/json"
+            "/api/v1/ads/campaigns", data=json.dumps(self.campaign_data()),
+            content_type="application/json", HTTP_X_CSRFTOKEN="x" * 32,
         )
-        
+        self.assertEqual(response.status_code, 403)
+        self.assertFalse(AdCampaign.objects.exists())
+
+    def test_reversed_dates_rejected_on_create(self):
+        response = self.write("post", "ads/campaigns", self.campaign_data(
+            start_date=self.end.isoformat(), end_date=self.start.isoformat()
+        ))
+        self.assertEqual(response.status_code, 422)
+        self.assertIsInstance(response.json()["detail"], str)
+        self.assertFalse(AdCampaign.objects.exists())
+
+    def test_partial_date_updates_validate_resulting_campaign(self):
+        campaign = self.campaign()
+        for data in [
+            {"start_date": (self.end + timedelta(days=1)).isoformat()},
+            {"end_date": (self.start - timedelta(days=1)).isoformat()},
+            {"start_date": None}, {"end_date": None},
+            {"start_date": "invalid-date"},
+        ]:
+            with self.subTest(data=data):
+                response = self.write("put", f"ads/campaigns/{campaign.id}", data)
+                self.assertEqual(response.status_code, 422)
+                campaign.refresh_from_db()
+                self.assertEqual(campaign.start_date, self.start)
+                self.assertEqual(campaign.end_date, self.end)
+
+    def test_equal_dates_allowed_on_create_and_update(self):
+        response = self.write("post", "ads/campaigns", self.campaign_data(end_date=self.start.isoformat()))
         self.assertEqual(response.status_code, 200)
-        self.assertEqual(AdBanner.objects.count(), 1)
-        
-        # List Banners
-        response = self.client.get("/api/v1/ads/banners")
+        path = f"ads/campaigns/{response.json()['id']}"
+        response = self.write("put", path, {"start_date": self.end.isoformat(), "end_date": self.end.isoformat()})
         self.assertEqual(response.status_code, 200)
-        data = response.json()
-        self.assertEqual(len(data), 1)
-        self.assertEqual(data[0]["name"], "Banner 1")
 
-    def test_list_brands(self):
-        response = self.client.get("/api/v1/core/brands")
+    def test_active_banner_requires_image(self):
+        campaign = self.campaign()
+        for changes in [{"image_id": None}, {"image_id": None, "is_active": True}]:
+            with self.subTest(changes=changes):
+                response = self.write("post", "ads/banners", self.banner_data(campaign, **changes))
+                self.assertEqual(response.status_code, 422)
+        self.assertFalse(AdBanner.objects.exists())
+
+    def test_banner_updates_validate_resulting_state(self):
+        campaign = self.campaign()
+        banner = AdBanner.objects.create(campaign=campaign, is_active=False)
+        path = f"ads/banners/{banner.id}"
+        self.assertEqual(self.write("put", path, {"is_active": True}).status_code, 422)
+        banner.refresh_from_db()
+        self.assertFalse(banner.is_active)
+        self.assertEqual(self.write("put", path, {"is_active": True, "image_id": str(self.image.id)}).status_code, 200)
+        self.assertEqual(self.write("put", path, {"image_id": None}).status_code, 422)
+        banner.refresh_from_db()
+        self.assertTrue(banner.is_active)
+        self.assertEqual(banner.image_id, self.image.id)
+        self.assertEqual(self.write("put", path, {"is_active": False, "image_id": None}).status_code, 200)
+        banner.refresh_from_db()
+        self.assertFalse(banner.is_active)
+        self.assertIsNone(banner.image_id)
+
+    def test_inactive_banner_without_image_allowed(self):
+        response = self.write("post", "ads/banners", self.banner_data(self.campaign(), image_id=None, is_active=False))
         self.assertEqual(response.status_code, 200)
-        data = response.json()
-        self.assertTrue(len(data) >= 1)
-        self.assertEqual(data[0]["name"], "Test Brand")
+        self.assertIsNone(response.json()["image_id"])
+
+    def test_scripts_rejected_on_create_and_update(self):
+        campaign = self.campaign()
+        script = "<script>alert(1)</script>"
+        response = self.write("post", "ads/banners", self.banner_data(campaign, script_code=script))
+        self.assertEqual(response.status_code, 422)
+        self.assertFalse(AdBanner.objects.exists())
+        banner = AdBanner.objects.create(campaign=campaign, image=self.image)
+        response = self.write("put", f"ads/banners/{banner.id}", {"script_code": script})
+        self.assertEqual(response.status_code, 422)
+        banner.refresh_from_db()
+        self.assertEqual(banner.script_code, "")
+
+    def test_missing_resources_rejected(self):
+        missing = uuid4()
+        for resource in ["campaigns", "banners"]:
+            path = f"ads/{resource}/{missing}"
+            self.assertEqual(self.client.get("/api/v1/" + path).status_code, 404)
+            self.assertEqual(self.write("put", path, {}).status_code, 404)
+            self.assertEqual(self.write("delete", path).status_code, 404)
+        self.assertEqual(self.write("post", "ads/campaigns", self.campaign_data(brand_id=str(missing))).status_code, 404)
+        campaign = self.campaign()
+        self.assertEqual(self.write("post", "ads/banners", self.banner_data(campaign, campaign_id=str(missing))).status_code, 404)
+        self.assertEqual(self.write("post", "ads/banners", self.banner_data(campaign, image_id=str(missing))).status_code, 404)
+        self.assertFalse(AdBanner.objects.exists())
+
+    def test_salesperson_flow_with_image_upload(self):
+        response = self.write("post", "core/brands", {"name": "New advertiser"})
+        self.assertEqual(response.status_code, 200)
+        brand_id = response.json()["id"]
+        response = self.write("post", "ads/campaigns", self.campaign_data(brand_id=brand_id))
+        self.assertEqual(response.status_code, 200)
+        campaign = AdCampaign.objects.get(id=response.json()["id"])
+        # Valid 1x1 GIF; file storage stays in memory.
+        image = SimpleUploadedFile(
+            "banner.gif",
+            b"GIF89a\x01\x00\x01\x00\x80\x00\x00\xff\xff\xff\x00\x00\x00!\xf9\x04\x01\x00\x00\x00\x00,\x00\x00\x00\x00\x01\x00\x01\x00\x00\x02\x02D\x01\x00;",
+            content_type="image/gif",
+        )
+        response = self.client.post("/api/v1/core/media", {"file": image}, HTTP_X_CSRFTOKEN=self.csrf)
+        self.assertEqual(response.status_code, 200)
+        image_id = response.json()["id"]
+        self.assertTrue(response.json()["file"])
+        response = self.write("post", "ads/banners", self.banner_data(campaign, image_id=image_id))
+        self.assertEqual(response.status_code, 200)
+        path = f"ads/banners/{response.json()['id']}"
+        response = self.client.get("/api/v1/" + path)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.json()["image_id"], image_id)
+        self.assertEqual(self.write("put", path, {"name": "Edited creative"}).status_code, 200)
+        response = self.client.get("/api/v1/" + path)
+        self.assertEqual(response.status_code, 200)
+        self.assertEqual(response.json()["name"], "Edited creative")
+        self.assertEqual(response.json()["image_id"], image_id)
```
