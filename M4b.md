**Start with H0: make the sales dashboard usable.** The code supports Abel’s assessment: the advertising backoffice exists, but the path from advertiser creation to a live, measurable banner is incomplete.

I reviewed the attached mail, [PLAN_MONETIZACION.md](/home/ubuntu/m4b/docs/PLAN_MONETIZACION.md), the advertising backend, dashboard forms, all three portal banner services, and running services. No files were changed.

| Area | What the code currently does | Required next work |
|---|---|---|
| Advertisers | Only `GET /core/brands`; no dashboard advertiser page | Add CRUD and safe deletion |
| Campaign/banner editing | Both edit pages use synchronous `params.id`; frontend uses Next 16.1.3 | Read params with `use(params)` |
| Form validation | No API date-order check or active-banner image requirement; script input enabled | Add validation and disable scripts |
| Requests | Forms already check `res.ok`, but lack shared CSRF handling | Centralize authenticated mutations |
| Inventory | Banners use generic `position`; no `slot_id` | Build the real slot catalog in H1 |
| Public delivery | All three `getAdvertisements()` implementations return `[]` | Implement delivery and connect all portals in H2 |
| Measurement/reporting | `AdMetric` exists, but no code writes advertising metrics or exports reports | Complete H3 and H4 |
| Operations | Worker and beat are defined but absent from running services; no healthchecks | Complete H5 operations work |

**Your next four working days should follow H0.1–H0.5 in order:**

1. **13 October — Advertiser CRUD.** Extend [core API](/home/ubuntu/m4b/backend/core/api/views.py), add `/dashboard/ads/advertisers`, and add the sidebar entry. Assign advertisers to the current organization. Guard deletion before calling `.delete()`: campaigns currently cascade from the advertiser.
2. **14 October — Repair editing.** Fix campaign and banner params, then verify opening an existing UUID and saving changes.
3. **15 October — Complete form behavior.** Add shared `apiFetch` with CSRF for saves, uploads and deletes. Validate dates against the resulting campaign on updates as well as creates. Require an image for active banners, disable the script field, and block campaign saving when no advertisers exist.
4. **16 October — Contract tests and demo.** Replace the anonymous ads tests with authenticated tests, including CSRF enforcement and rejection cases. Run `python manage.py test ads core`, then demonstrate advertiser → campaign → uploaded banner → reopen → edit.

Put endpoint tests in the same changes as the endpoints; H0.5 should consolidate coverage rather than postpone it.

**Three implementation details need attention beyond the mail’s summary:**

- **Portal targeting still uses Frappe labels.** `Banner.tsx` passes `frappePlacement` into the hook. H2 must query the real `placement`, such as `home-sidebar-top`, and the actual portal domain.
- **The banner hook caches for five minutes.** That conflicts with the 60-second activation/pause requirement. Update client caching and refresh behavior alongside server caching; a server TTL alone won’t fix it.
- **Random selection happens on every render.** Stabilize the selected creative before adding visibility tracking so impressions and clicks refer to the banner the reader actually saw.

Then continue with the agreed milestones:

| Milestone | Deliverable | Demo |
|---|---|---|
| H1 | Real slots, size validation, inventory screen | 22 October |
| H2 | Live banners across all three portals; safe click redirect | 29 October |
| H3 | Visible impressions, counted clicks, bot filtering | 5 November |
| H4 | Campaign results and CSV | 10 November |
| H5 | Consent, operations, complete salesperson flow | 13 November |

Use the mail’s exact demo dates; the local plan lists durations but no dated calendar. Keep SaaS, Stripe, multi-tenancy and Frappe integration deferred as instructed.

The checkout is on `main` with existing uncommitted changes, including core API and Compose changes. Preserve and identify those before starting H0, then use the preferred feature-branch/MR workflow in [git-deploy-main.md](/home/ubuntu/m4b/docs/git-deploy-main.md). Work through the corresponding ERP tickets in order; I haven’t inspected or updated those tickets.

**H0 is complete when a salesperson can create and edit an advertiser, campaign and image banner without developer help.** Live delivery belongs to H2. Tests and live-domain acceptance remain to be performed; this review confirms the implementation gaps, not milestone completion.
