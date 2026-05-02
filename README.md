# Geo-Intelligent Auto Catalog System

## Scope

Use this file as the single implementation guide for:

- `backend/catalogue_mgmt_service`
- `frontend/hha_web`

Note:

- `hha_web` is Angular/Ionic in this repo, not React.
- Backend local env file in this repo is `backend/catalogue_mgmt_service/.lcl.env`, not `lcl.dev`.

---

## Step 0: Actual Code Trace Summary

### 1. Shop -> pincode mapping

In `catalogue_mgmt_service`, shop geo fields are not stored in `retailercatalog`.

Actual existing source traced in code:

- `src/apis/controllers/v1/retailerCatalog/index.js`
- function: `getShopDataShopIds(shopIds)`
- calls: `GET ${loadBalancer}/rms/apis/v2/shop/getPGShopDetails/${shopIds}`

Fields used there:

- `shop_id`
- `shop_name`
- `city`
- `locality`
- `pincode`
- `street`
- `user_id`
- `selling_type`

Related geo helper already exists:

- `src/apis/controllers/v1/retailerCatalog/utils/apiCalls.js`
- function: `getAllShopLatLon()`
- calls: `GET ${loadBalancer}/rms/apis/v1/shop/allShopLatLon`

### 2. How `retailercatalog` is fetched

Mongo model:

- `src/apis/models/mongoCatalog/retailerSchema.js`
- collection/model: `retailercatalog`

Service used:

- `src/apis/services/v1/mongoCatalog/retailerCatalog.js`

Main fetch methods:

- `getProductByShopId({ shopId })`
- `getProductByProductId({ shopId, productId })`
- `getRetailerAllTypesOfCatalogByshopIds(shopIds)`

Main route:

- `src/apis/routes/v1/retailerCatalog/index.js`
- `GET /retailercatalog/getRetailerCatalog/:shopId`

Controller:

- `src/apis/controllers/v1/retailerCatalog/index.js`
- function: `getRetailerCatalog({ shopId })`

Important stored product field:

- `retailercatalog.catalog.prdNm`

Other useful retailer fields:

- `retailercatalog.shopId`
- `retailercatalog.category`
- `retailercatalog.catalog.catPnm`
- `retailercatalog.catalog.catPid`

### 3. Existing catalog APIs

Backend route index:

- `src/apis/routes/v1/index.js`

Existing catalog-related routes:

- `/catalog`
- `/category`
- `/product`
- `/customCatalog`
- `/retailercatalog`

Important routes:

- `GET /retailercatalog/getRetailerCatalog/:shopId`
- `POST /retailercatalog/addTopProducts/:shopId`
- `POST /retailercatalog/addProductsToRetailer`
- `PUT /retailercatalog/updateRetailerProducts/:shopId`
- `GET /customCatalog/getCustomCatalogRequests/:shopId`
- `POST /customCatalog`

### 4. How frontend consumes APIs

`hha_web` does not call CMS directly for retail catalog. It consumes store metadata and profile JSON URLs.

Main frontend service:

- `frontend/hha_web/src/app/lib/services/catalogue.service.ts`

Main API wrapper:

- `frontend/hha_web/src/app/lib/services/api/common.api.ts`

Environment:

- `frontend/hha_web/src/environments/environment.ts`

Current frontend catalog flow:

1. Fetch store metadata:
   - `ApiUrls.storeMeta = /hms/apis/v1/households/storeAllData`
2. Read profile JSON URL from `store_meta[0].url`
3. Fetch catalog JSON from CDN/profile URL
4. Render in:
   - `src/app/pages/store/productlisting/productlisting.page.ts`
   - `src/app/pages/store/store-details/store-details.page.ts`

---

## Step 1: Local Env Setup

Use:

- `backend/catalogue_mgmt_service/.lcl.env`

Set:

```env
SQL_DB_HOST=localhost
SQL_DB_USER=postgres
SQL_DB_PASSWORD=omkar
SQL_DB_PORT=5432
SQL_DB_NAME=cms

MONGO_DB_HOST=mongodb://127.0.0.1:27017/metadata
```

Also make sure local-only fallback values exist if required by config:

```env
QUEUE_URL=http://localhost/queue
ENVIRONMENT=dev
MEDIA_S3=http://localhost/media
TOP_PRODUCTS_LIMIT=10
RADIUS=25
CITY_LAT_LON=[{"city":"Bengaluru","lat":12.9716,"lon":77.5946}]
```

If PostgreSQL SSL blocks startup, keep SQL config without SSL since knex config here only uses host/user/password/port/database.

Run:

```bash
cd backend/catalogue_mgmt_service
npm install
npm run lcl
```

If `nodemon` sandbox blocks, direct runtime equivalent is:

```bash
node -r dotenv/config ./server dotenv_config_path=./.lcl.env
```

---

## Step 2: Required Data Sources

Primary:

- `retailercatalog`

Secondary:

- `products`
- `customcatalog`
- `categories`

Actual model locations:

- `src/apis/models/mongoCatalog/retailerSchema.js`
- `src/apis/models/mongoCatalog/productSchema.js`
- `src/apis/models/mongoCatalog/customCatalogSchema.js`
- `src/apis/models/mongoCatalog/categorySchema.js`

Important product name field:

- `retailercatalog.catalog.prdNm`

Category source priority:

1. `retailercatalog.category`
2. `retailercatalog.catalog.catPnm`
3. `products.catPnm`

---

## Files To Create

### Backend

Create:

- `backend/catalogue_mgmt_service/src/apis/utils/normalizeProduct.js`
- `backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/geoCatalogSchema.js`
- `backend/catalogue_mgmt_service/src/apis/services/v1/pincodeCatalogBuilder.service.js`
- `backend/catalogue_mgmt_service/src/apis/services/v1/geoHierarchy.service.js`
- `backend/catalogue_mgmt_service/src/apis/services/v1/ai.service.js`
- `backend/catalogue_mgmt_service/src/apis/controllers/v1/geo.js`
- `backend/catalogue_mgmt_service/src/apis/routes/v1/geo.js`
- `backend/catalogue_mgmt_service/src/jobs/geoCatalog.job.js`

Update:

- `backend/catalogue_mgmt_service/src/apis/routes/v1/index.js`
- `backend/catalogue_mgmt_service/src/InitApp/index.js`
- `backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/retailerSchema.js`
- `backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/customCatalogSchema.js`

### Frontend

Create:

- `frontend/hha_web/src/app/lib/services/geo-catalog.service.ts`
- `frontend/hha_web/src/app/pages/geo-catalog-test/*`

Update:

- `frontend/hha_web/src/app/app-routing.module.ts`

---

## Backend Implementation Contract

### 1. Normalizer

File:

- `src/apis/utils/normalizeProduct.js`

Rules:

- lowercase
- remove punctuation
- remove units: `ml`, `g`, `kg`, `l`
- trim spaces

Example:

```js
"Amul Gold Milk 500ml" -> "amul gold milk"
```

### 2. Geo catalog schema

File:

- `src/apis/models/mongoCatalog/geoCatalogSchema.js`

Collection:

- `geo_catalogs`

Document:

```js
{
  level: "PINCODE",
  pincode: "560100",
  city: "Bengaluru",
  state: "Karnataka",
  country: "India",
  categories: [
    {
      name: "dairy",
      products: [
        { name: "amul gold milk", count: 120 },
        { name: "nandini milk", count: 90 }
      ]
    }
  ],
  createdAt,
  updatedAt
}
```

### 3. Pincode builder service

File:

- `src/apis/services/v1/pincodeCatalogBuilder.service.js`

Function:

```js
buildPincodeCatalog(pincode)
```

Logic:

1. resolve shops for pincode `560100`
2. fetch `retailercatalog` where `shopId in shops`
3. extract:
   - name from `catalog.prdNm`
   - category from `category` or first segment of `catPnm`
4. normalize names
5. group by category
6. frequency count by normalized product
7. sort desc by count
8. keep top `10-20`
9. store one document in `geo_catalogs`
10. if low data, write fallback category/product set

### 4. Hierarchy service

File:

- `src/apis/services/v1/geoHierarchy.service.js`

Functions:

- resolve with fallback:
  - `PINCODE`
  - `CITY`
  - `STATE`
  - `COUNTRY`
- apply geo catalog to shop

Fallback order:

```txt
pincode -> city -> state -> country
```

### 5. Mock AI service

File:

- `src/apis/services/v1/ai.service.js`

Only return:

- normalized names helper
- fallback products by category

No external API call.

### 6. Geo controller

File:

- `src/apis/controllers/v1/geo.js`

Methods:

- `testPincodeCatalog`
- `applyGeoCatalog`
- optional `getResolvedGeoCatalog`

### 7. Geo routes

File:

- `src/apis/routes/v1/geo.js`

Routes:

```txt
POST /geo/test/pincode
POST /geo/apply
GET  /geo/catalog
```

Body for test:

```json
{
  "pincode": "560100"
}
```

Expected response:

```json
{
  "success": true,
  "categories": [
    {
      "name": "dairy",
      "products": [
        { "name": "amul gold milk", "count": 120 }
      ]
    }
  ]
}
```

### 8. Cron job

File:

- `src/jobs/geoCatalog.job.js`

Run time:

- daily `2 AM`

Process:

1. fetch all shops
2. group by pincode
3. build all pincode catalogs
4. rebuild city/state/country catalogs

Wire startup from:

- `src/InitApp/index.js`

### 9. Apply to shop

Route:

- `POST /geo/apply`

Body:

```json
{
  "shopId": 1234,
  "pincode": "560100"
}
```

Logic:

1. resolve geo catalog with fallback
2. insert products into `customcatalog`
3. set:

```txt
productNameStatus = "UNVERIFIED"
```

Recommended insert shape in `customcatalog`:

```js
{
  shopId,
  retailerId,
  guid,
  retailerName: "Geo Catalog Auto Apply",
  productId: "geo-PINCODE-dairy-amul gold milk",
  productName: "amul gold milk",
  productNameStatus: "UNVERIFIED",
  productDescriptionStatus: "NEW",
  productImageStatus: "NEW",
  description: "Auto-added from PINCODE geo catalog",
  category: "dairy",
  subCategory: "dairy",
  updateStatus: "NEW"
}
```

You may need to extend enums in:

- `src/apis/models/mongoCatalog/customCatalogSchema.js`

to allow `UNVERIFIED`.

---

## Indexes

Add:

### `retailercatalog`

File:

- `src/apis/models/mongoCatalog/retailerSchema.js`

```js
retailer_Catalog.index({ shopId: 1 });
```

### `geo_catalogs`

File:

- `src/apis/models/mongoCatalog/geoCatalogSchema.js`

At minimum:

```js
geoCatalogSchema.index({ level: 1, pincode: 1 }, { sparse: true });
```

---

## Frontend Integration

This repo’s frontend is Angular, so use Angular files.

### API service

File:

- `frontend/hha_web/src/app/lib/services/geo-catalog.service.ts`

Method:

```ts
runGeoTest() {
  return this.http.post('http://localhost:2210/cms/apis/geo/test/pincode', {
    pincode: '560100'
  });
}
```

### Test page

Create:

- `frontend/hha_web/src/app/pages/geo-catalog-test/`

Show:

- category name
- top products
- counts

Add route in:

- `frontend/hha_web/src/app/app-routing.module.ts`

Suggested route:

```txt
/geo-catalog-test
```

---

## Validation Flow

### Backend

```bash
cd backend/catalogue_mgmt_service
npm install
npm run lcl
```

### API test

```http
POST http://localhost:2210/cms/apis/geo/test/pincode
Content-Type: application/json

{
  "pincode": "560100"
}
```

Check:

- multiple categories
- multiple products per category

### Mongo

Verify collection:

- `geo_catalogs`

Verify document:

- one doc for pincode
- category array exists
- product arrays exist

### Apply

```http
POST http://localhost:2210/cms/apis/geo/apply
Content-Type: application/json

{
  "shopId": 1234,
  "pincode": "560100"
}
```

Check:

- entries created in `customcatalog`
- `productNameStatus = "UNVERIFIED"`

---

## Known Local Runtime Notes

If local startup fails, common repo-specific issues are:

1. missing dependency install in `backend/catalogue_mgmt_service`
2. local Mongo not running on `127.0.0.1:27017`
3. `npm run lcl` using `nodemon` may fail under some restricted shells
4. newer Node versions can break older transitive libs in this repo

For local work, prefer first verifying:

- Mongo running on `27017`
- Postgres running on `5432`
- `.lcl.env` loaded

---

## Minimal Backend File List To Touch

```txt
backend/catalogue_mgmt_service/.lcl.env
backend/catalogue_mgmt_service/src/InitApp/index.js
backend/catalogue_mgmt_service/src/apis/routes/v1/index.js
backend/catalogue_mgmt_service/src/apis/routes/v1/geo.js
backend/catalogue_mgmt_service/src/apis/controllers/v1/geo.js
backend/catalogue_mgmt_service/src/apis/services/v1/pincodeCatalogBuilder.service.js
backend/catalogue_mgmt_service/src/apis/services/v1/geoHierarchy.service.js
backend/catalogue_mgmt_service/src/apis/services/v1/ai.service.js
backend/catalogue_mgmt_service/src/apis/utils/normalizeProduct.js
backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/geoCatalogSchema.js
backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/retailerSchema.js
backend/catalogue_mgmt_service/src/apis/models/mongoCatalog/customCatalogSchema.js
backend/catalogue_mgmt_service/src/jobs/geoCatalog.job.js
```

## Minimal Frontend File List To Touch

```txt
frontend/hha_web/src/app/app-routing.module.ts
frontend/hha_web/src/app/lib/services/geo-catalog.service.ts
frontend/hha_web/src/app/pages/geo-catalog-test/geo-catalog-test-routing.module.ts
frontend/hha_web/src/app/pages/geo-catalog-test/geo-catalog-test.module.ts
frontend/hha_web/src/app/pages/geo-catalog-test/geo-catalog-test.page.ts
frontend/hha_web/src/app/pages/geo-catalog-test/geo-catalog-test.page.html
frontend/hha_web/src/app/pages/geo-catalog-test/geo-catalog-test.page.scss
```
