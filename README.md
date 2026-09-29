# Project 1 — RAP Product & Stock Movement Service (OData V4)

## Problem Statement
Warehouse/retail teams need a simple way to track product master data
alongside every stock adjustment (inbound, outbound, corrections) without
losing an audit trail of who changed what and when. This project builds a
**draft-enabled, CRUD-Q RAP business service** on SAP BTP ABAP Environment
that manages `Product` records and their associated `Stock Movements`,
exposes them as an OData V4 service, and auto-generates a Fiori Elements
UI on top — with a custom `reorderNow` action and server-side stock
validation baked into the business layer.

## Architecture

```mermaid
erDiagram
    PRODUCT ||--o{ STOCKMOVEMENT : has
    PRODUCT {
        uuid product_id PK
        string product_name
        int stock_qty
        int reorder_level
        timestamp last_changed_at
    }
    STOCKMOVEMENT {
        uuid movement_id PK
        uuid product_id FK
        int qty_change
        date movement_date
    }
```

**RAP layer stack**

```mermaid
flowchart LR
    DB1[(z_product<br/>Persistent Table)]
    DB2[(z_product_d<br/>Draft Table)]

    DB1 -->|active data| ROOT[Z_PRODUCT_I<br/>Root View Entity<br/>@Draft.enabled: true]
    DB2 -->|draft shadow data| ROOT

    ROOT -->|exposed via| CV[Z_PRODUCT<br/>Consumption View]
    ROOT -->|implements| BDEF[Z_PRODUCT_I.bdef<br/>managed, strict 2<br/>etag master, lock master<br/>authorization master<br/>instance]

    CV --> SB[Service Binding<br/>OData V4 / Web API]
    SB --> Fiori[Fiori Elements UI]

    BDEF --> CRUD[create / update / delete<br/>field readonly: product_id,<br/>last_changed_at]

    BDEF --> DRAFT[draft actions:<br/>Edit / Activate / Discard / Resume<br/>draft determine action: Prepare]

    BDEF --> VAL[validation: validateStock<br/>on save]
    BDEF --> DET[determination: setLastChanged<br/>on save, on create]
    BDEF --> ACT[action: reorderNow]

    BDEF -->|managed implementation<br/>in class| GCS["Global Class Shell<br/>(empty, CREATE PRIVATE)"]

    GCS -->|Local Definitions /<br/>Implementations include| LHC[Local Handler Class<br/>lhc_zrt_product<br/>INHERITING FROM<br/>cl_abap_behavior_handler]

    LHC --> VS[validateStock<br/>FOR VALIDATE ON SAVE]
    LHC --> SL[setLastChanged<br/>FOR DETERMINE ON SAVE]
    LHC --> RN[reorderNow<br/>FOR MODIFY / ACTION]
    LHC --> IA[get_instance_authorizations<br/>FOR INSTANCE AUTHORIZATION]

    VS -->|executes| VAL
    SL -->|executes| DET
    RN -->|executes| ACT
```

## Tech Stack
- **SAP BTP ABAP Environment** (Steampunk) — RAP (RESTful ABAP Programming Model)
- **CDS Views** — interface + projection layers with `@UI` annotations
- **Behavior Definitions/Implementations** — managed, draft-enabled
- **OData V4** service binding, tested via **Swagger UI** and Postman
- **SAP Fiori Elements** — auto-generated List Report / Object Page

## What Was Built
| Layer | Object | Purpose |
|---|---|---|
| Database Table | `zrr_product` | Product master (`product_id`, `product_name`, `stock_qty`, `reorder_level`, `last_changed_at`) |
| Database Table | `zrr_stockmovem` | Stock movement log (`movement_id`, `product_id`, `qty_change`, `movement_date`) |
| Interface View | `ZRT_PRO_IV` | Root view, composition to `_Movements` |
| Interface View | `ZRT_STOCKMOVEMENT_IV` | Child view, association to parent `_Product` |
| Projection View | `ZRT_PRO_PV` | UI-annotated projection (`@UI.lineItem`, `@UI.identification`, `@UI.selectionField`, `@UI.facet`) |
| Projection View | `ZRT_STOCKMOVEMENT_PV` | Redirected composition child projection |
| Behavior Definition | `ZRT_PRO_PV` (managed, strict(2), draft) | create/update/delete, validation, determination, custom action |
| Service Definition | `ZRT_PRODUCT_SD` | Exposes `Product { _Movements }` |
| Service Binding | `ZRT_PRO_IV_SB` (OData V4 - UI) | Published service + Fiori App URL |

### Business Logic
- **Validation** `validateStock` — runs on save, prevents negative stock quantities.
- **Determination** `setLastChanged` — auto-stamps `last_changed_at` on create/update.
- **Custom Action** `reorderNow` — bumps stock by a fixed reorder quantity via a single call.
- **Draft handling** — `with draft`, `use action Edit/Resume/Activate/Discard/Prepare` — every create/edit goes through a draft before activation, matching standard Fiori Elements UX.

## Screenshots

| | |
|---|---|
| ![DB tables: zrr_product, zrr_stockmovem](project-1-rap-odata/docs/images/01-db-tables.png) | Database tables `ZRR_PRODUCT` and `ZRR_STOCKMOVEM` with key fields |
| ![Interface views with association/composition](project-1-rap-odata/docs/images/02-interface-views.png) | `ZRT_STOCKMOVEMENT_IV` and `ZRT_PRO_IV` — composition + association |
| ![Projection view with UI annotations](project-1-rap-odata/docs/images/03-projection-ui-annotations.png) | `ZRT_PRO_PV` — `@UI.headerInfo`, `@UI.facet`, `@UI.lineItem`, `@UI.selectionField` |
| ![Behavior definition — managed, draft, validation, action](project-1-rap-odata/docs/images/04-behavior-definition.png) | `ZRT_PRO_PV`/`ZRT_STOCKMOVEMENT_PV` — validations, determinations, `reorderNow` action, draft actions |
| ![Service binding published, entity set preview](project-1-rap-odata/docs/images/05-service-binding-published.png) | `ZRT_PRO_IV_SB` — OData V4-UI binding, `Product`/`Movement` entity sets, Fiori App URL |
| ![Swagger metadata for the service](project-1-rap-odata/docs/images/06-swagger-metadata.png) | Auto-generated Swagger UI — `GET /Product`, `GET /Product/{product_id}`, `POST /$batch` |
| ![Fiori Elements list report — Products](project-1-rap-odata/docs/images/07-fiori-list-report.png) | Live Fiori Elements list report: Products with `stock_qty`, `reorder_level` |
| ![Fiori Elements list report — Stock Movements](project-1-rap-odata/docs/images/08-fiori-stock-movements.png) | Associated Stock Movements list, driven by the `_Movements` composition |
| ![New Object dialog — Time Stamp picker](project-1-rap-odata/docs/images/09-create-new-object-dialog.png) | Create form with date/time picker for `last_changed_at` |
| ![Draft entry before save](project-1-rap-odata/docs/images/10-create-draft-entry.png) | Draft state — **Create** / **Discard Draft** actions |
| ![Object created confirmation](project-1-rap-odata/docs/images/11-create-object-created.png) | Draft activated — **Object created** toast, Edit/Delete now available |
| ![List refreshed after create](project-1-rap-odata/docs/images/12-list-after-create.png) | Products list showing the new row after activation |
| ![Delete flow](project-1-rap-odata/docs/images/13-delete-flow.png) | Multi-select delete → **Objects deleted** toast |

> Save your screenshots into `docs/images/` with these exact names (or update
> the paths above) — see the "Presenting this on GitHub" notes below for why.

## API Contract

| Operation | Method | Endpoint | Notes |
|---|---|---|---|
| List products | `GET` | `/Product` | Returns all products |
| Read one product | `GET` | `/Product('<uuid>')` | By key |
| Create product | `POST` | `/Product` | Creates a draft first (draft-enabled) |
| Update product | `PATCH` | `/Product('<uuid>')` | `product_id` and `last_changed_at` are read-only |
| Delete product | `DELETE` | `/Product('<uuid>')` | |
| Reorder stock | `POST` | `/Product('<uuid>')/reorderNow` | Custom RAP action, bumps `stock_qty` |
| List stock movements | `GET` | `/Movement` | Or via `/Product('<uuid>')/_Movements` |
| Batch requests | `POST` | `/$batch` | Groups multiple operations in one call |

## What I'd Do Differently at Scale
- Move `validateStock` and `setLastChanged` logic into a shared, reusable
  class rather than embedding it directly in the behavior implementation,
  so the same rules can be applied across multiple behavior definitions.
- Add authorization checks at the instance level (currently
  `#NOT_REQUIRED` on the interface views) before this goes anywhere near
  production.
- Introduce numbering/ID generation via a proper number range object
  instead of relying solely on managed UUID keys, if this needs to
  integrate with external systems that expect sequential IDs.
