# PPX PO Gen - Automated Accounts Payable System - Technical Documentation

## 📋 Table of Contents

1.  [System Architecture](#system-architecture)
2.  [Backend Components](#backend-components)
3.  [OCR Service Components](#ocr-service-components)
4.  [Frontend Components](#frontend-components)
5.  [Data Flow & API Endpoints](#data-flow--api-endpoints)
6.  [Component Logic Explanation](#component-logic-explanation)
7.  [AI Integration (Perplexity OCR)](#ai-integration-perplexity-ocr)
8.  [Local Development Setup](#local-development-setup)
9.  [Testing & Deployment](#testing--deployment)
10. [Conclusion](#conclusion)

 

## 🏗️ System Architecture

```

                ┌─────────────────────────────────────────────────────────────────────────────┐
                │                           Frontend (Vue 3 / Vite)                           │
                │   ┌─────────────┐ ┌───────────┐  ┌─────────┐ ┌─────────┐ ┌────────────────┐ │
                │   │ Auth Pages  │ │ Dashboard │  │ Invoices│ │ Vendors │ │ PO Mgmt        │ │
                │   │(Login/Reg)  │ │           │  │ (Upload)│ │         │ │(List/Edit/View)│ │
                │   └─────────────┘ └───────────┘  └─────────┘ └─────────┘ └──────┬─────────┘ │
                └───────────────────────────────────┬─────────────────────────────────│───────┘
                                                    │ HTTP/JSON API (Axios)           │ PDF/Word Gen
                ┌───────────────────────────────────┴─────────────────────────────────▼────────┐
                │                        Backend (Node.js / Express)                           │
                │   ┌──────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌────────┐  │
                │   │ Auth API │ │ Invoice │ │ Vendor  │ │  PO API │ │ File Gen  │ │ Multer │  │
                │   │ (JWT)    │ │ API     │ │ API     │ │ (CRUD)  │ │(PDF/Word) │ │ Upload │  │
                │   └─────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └─────┬─────┘ └────▲───┘  │
                │         │           │           │           │            │            │      │
                │         └───────────▼───────────▼───────────▼────────────▼────────────┘      │
                │                        Models (Mongoose) & DB Connection                     │
                └──────────────┬────────────────────┬─────────────────────────────┬────────────┘
                               │ DB Read/Write      │ Job Push (LPUSH)            │ File Path
                ┌──────────────▼────────────────────▼─────────┐   ┌───────────────▼─────────────┐
                │              MongoDB                        │   │         Redis               │
                │   (Users, Invoices, Vendors, POs)           │   │   (Job Queue: Processing)   │
                └─────────────────────────────────────────────┘   └──────────────┬──────────────┘
                │ Job Pop (BRPOP)
                ┌────────────────▼────────────────┐
                │ OCR Service (Python / Redis-py) │
                │ ┌────────────┐ ┌────────────┐   │
                │ │ Perplexity │ │ File Read  │   │
                │ │ Client     │ │ (Uploads)  │   │
                │ └──────┬─────┘ └──────▲─────┘   │
                └────────│──────────────│─────────┘
                │              │ File Access
                │ API Call     │ (Read Only)
                ┌────────▼────────┐ ┌───▼───────────┐
                │ Perplexity API  │ │ Shared Volume │
                │ (OCR Extraction)│ │  (Uploads)    │
                └─────────────────┘ └───────────────┘

````
 

### Technology Stack

Backend:
- Node.js: JavaScript runtime environment
- Express: Web application framework for Node.js
- Mongoose: MongoDB object modeling tool
- MongoDB: NoSQL database for storing user, invoice, vendor, and PO data
- Redis: In-memory data structure store, used as a message queue
- ioredis: Redis client for Node.js
- JSON Web Token (JWT): For user authentication
- bcryptjs: For password hashing
- Multer: Middleware for handling `multipart/form-data` (file uploads)
- PDFKit: PDF generation library
- docx: DOCX (Word document) generation library
- dotenv: Loads environment variables from a `.env` file
- cors: Middleware for enabling Cross-Origin Resource Sharing

OCR Service:
- Python 3.9: Programming language
- Redis-py: Redis client for Python
- Pymongo: MongoDB driver for Python
- Requests: HTTP library for calling the Perplexity API
- python-dotenv: Loads environment variables

Frontend:
- Vue 3: Progressive JavaScript framework
- Vite: Frontend build tool and development server
- Vue Router: Official router for Vue.js
- Tailwind CSS: Utility-first CSS framework
- Axios: Promise-based HTTP client for making API requests
- html2canvas: Library for taking "screenshots" of HTML elements (used for PNG download)

Containerization:
- Docker: Platform for developing, shipping, and running applications in containers
- Docker Compose: Tool for defining and running multi-container Docker applications

 

## 🔧 Backend Components

### 1. Main Application (`server.js`)

Purpose: Entry point for the Node.js backend application. Initializes Express, connects to the database, sets up middleware, and defines API routes.
Key Features:
- Environment Loading: Uses `dotenv` to load configuration from `.env`.
- Database Connection: Calls `connectDB` from `src/config/db.js`.
- Middleware: Includes `cors`, `express.json`, `express.urlencoded`.
- Routing: Mounts various route handlers (`auth`, `invoices`, `vendors`, `purchase-orders`, `generate`) under the `/api` prefix.
- Health Check: Provides a basic `/api/health` endpoint.
- Port Configuration: Reads `PORT` from environment variables, defaulting to 5002.

### 2. Database Configuration (`src/config/db.js`)

Purpose: Handles connections to MongoDB and Redis.
Key Features:
- MongoDB Connection: Uses Mongoose to connect to the MongoDB URI specified in `MONGO_URI`. Includes basic error handling and logs connection status. Exits the process on connection failure.
- Redis Connection: Uses `ioredis` to connect to Redis using `REDIS_HOST` and `REDIS_PORT`. Includes configuration for retries (`retryStrategy`) and event listeners (`connect`, `ready`, `error`, etc.) to log Redis client status. Exports the `redisClient` instance.

### 3. Data Models (`src/models/`)

Purpose: Define Mongoose schemas for database collections.

- `User.js`: Schema for user accounts (name, email, password, registerDate). Includes an index on email.
- `Invoice.js`: Schema for uploaded invoices. Includes `invoiceNumber`, `status` (enum: Processing, Pending, Approved, Paid, Error), `extractedData` (nested schema matching Perplexity output structure), `filePath`, and timestamps. Contains sub-schemas for the expected Perplexity JSON structure. Includes `pre('save')` middleware for `updatedAt`.
- `Vendor.js`: Schema for vendors (name, email, phone, gstin, createdAt). Includes validation (name required, email format), indexing (name), and configuration for unique but sparse email.
- `PurchaseOrder.js`: Schema for Purchase Orders. Contains fields mapped from `Invoice.extractedData` and user edits (`poNumber`, `poDate`, `dueDate`, `orderBy`, `orderTo`, `shippingDetails`, `items`, `notes`, `terms`, `businessLogo`, `moreFields`), calculated financial fields (`subTotal`, `discount`, `taxableAmount`, `cgstTotal`, `sgstTotal`, `totalAmount`), `status`, a reference to the original `Invoice`, `templateName`, and timestamps. Includes sub-schemas for addresses, items, and moreFields. Includes middleware for `updatedAt`.

### 4. API Routes (`src/routes/`)

Purpose: Define API endpoints and link them to controller functions.

- `authRoutes.js`: Public routes for user registration (`/register`) and login (`/login`). Uses `authController`.
- `invoiceRoutes.js`: Private routes (require authentication) for invoice management. Includes file upload (`POST /upload`) using Multer, fetching all invoices (`GET /`), approving (`PUT /:id/approve`), and deleting (`DELETE /:id`). Configures Multer for disk storage with unique filenames and file filtering (images/PDFs). Uses `invoiceController` and `authMiddleware`.
- `vendorRoutes.js`: Private routes for vendor CRUD operations (`POST /`, `GET /`, `GET /:id`, `PUT /:id`, `DELETE /:id`). Uses `vendorController` and `authMiddleware`.
- `purchaseOrderRoutes.js`: Private routes for PO CRUD operations (`POST /`, `GET /`, `GET /:id`, `PUT /:id`, `DELETE /:id`). Uses `purchaseOrderController` and `authMiddleware`.
- `fileGenerationRoutes.js`: Private routes for downloading generated PO files. Includes PDF (`GET /pdf/:id`) and Word (`GET /word/:id`) generation. Uses a custom middleware (`authFromQueryOrHeader`) to allow authentication via URL query parameter (for direct download links) or standard header. Uses `fileGenerationController`.

### 5. Controllers (`src/controllers/`)

Purpose: Contain the business logic for handling API requests.

- `authController.js`: Handles user registration (hashing passwords with bcrypt, creating user, signing JWT) and login (finding user, comparing passwords, signing JWT).
- `invoiceController.js`:
    - `uploadInvoice`: Handles file uploads via Multer. Creates an `Invoice` record in MongoDB with 'Processing' status, saves the file path, and pushes a job (`{ invoiceId }`) to the Redis queue (`REDIS_QUEUE_NAME`). Includes cleanup logic for failed uploads.
    - `getAllInvoices`: Fetches all invoices, sorted by status and creation date.
    - `approveInvoice`: Updates an invoice's status to 'Approved'.
    - `deleteInvoice`: Deletes an invoice record and its associated file from the uploads directory. Checks if the invoice is linked to a PO before deletion.
- `vendorController.js`: Implements standard CRUD logic for vendors, including validation (name required, email uniqueness check).
- `purchaseOrderController.js`:
    - `createPO`: Creates a new PO based on data extracted from a processed `Invoice` (`invoiceId` required). Fetches the invoice, validates its status, maps `invoice.extractedData` to the `PurchaseOrder` schema (including parsing numbers and dates safely), calculates initial totals, and saves the new PO. Includes helper functions `parseSafeFloat` and `parseFlexibleDate` for robust data parsing.
    - `getPOById`, `updatePO`, `getAllPOs`, `deletePO`: Implement standard CRUD logic for POs. `updatePO` uses `findByIdAndUpdate` for atomicity and validation. `getAllPOs` populates linked invoice details.
- `fileGenerationController.js`:
    - `generatePDF`: Fetches a PO by ID and generates a PDF document using `PDFKit`. Dynamically lays out header, addresses, items table (handling page breaks), totals, notes, and terms. Includes helpers for currency/date formatting and safe text placement. Streams the PDF to the response.
    - `generateWord`: Fetches a PO by ID and generates a `.docx` document using the `docx` library. Builds the document structure programmatically (paragraphs, tables, images) based on PO data. Sends the generated buffer as the response.

### 6. Middleware (`src/middleware/authMiddleware.js`)

Purpose: Verifies JWT tokens for protected routes.
Logic: Extracts the token from the `x-auth-token` header, verifies it using `JWT_SECRET`, and attaches the decoded user payload (`req.user`) to the request object if valid. Returns 401 Unauthorized if the token is missing or invalid.

 

## ⚙️ OCR Service Components

### 1. Main Worker Script (`main.py`)

Purpose: Python service that listens to the Redis queue, processes invoice OCR jobs using the Perplexity API, and updates the invoice status in MongoDB.
Key Features:
- Environment Loading: Uses `python-dotenv` to load configuration (`MONGO_URI`, `REDIS_*`, `PERPLEXITY_*`).
- Database/Redis Connections: Functions (`connect_mongo`, `connect_redis`, `ensure_connections`) to establish and maintain connections using `pymongo` and `redis-py`, with retry logic.
- Perplexity API Interaction (`call_perplexity_ocr`):
    - Reads the invoice file (image or PDF) based on the path from MongoDB.
    - Encodes the file content to Base64.
    - Constructs a detailed prompt instructing the Perplexity API to extract specific fields in a strict JSON format.
    - Sends the prompt and Base64 image data to the `PERPLEXITY_API_ENDPOINT` using the `requests` library. Specifies the model (`sonar`) and `max_tokens` (1024).
    - Parses the JSON response from the API, attempting to find the structured data within common response patterns (e.g., `choices[0].message.content`). Includes logic to clean and validate the extracted JSON.
    - Handles API errors (e.g., 4xx/5xx status codes, connection issues, timeouts) and file errors.
- Invoice Processing (`process_invoice`):
    - Takes an `invoiceId` from the Redis job.
    - Fetches the corresponding `Invoice` document from MongoDB.
    - Validates the invoice status and file path.
    - Determines the file MIME type based on the extension.
    - Calls `call_perplexity_ocr`.
    - Calls `update_invoice_in_db` to update the invoice status to 'Pending' (with extracted data) or 'Error' (with an error message) based on the OCR result.
- Database Update (`update_invoice_in_db`): Updates the specified invoice document in MongoDB with the new status, extracted data, or error message. Includes checks for connection availability.
- Worker Loop (`main_worker_loop`):
    - Continuously listens to the Redis queue (`REDIS_QUEUE_NAME`) using blocking pop (`brpop`) with a timeout.
    - Parses the JSON job message to get the `invoiceId`.
    - Calls `process_invoice` for each valid job.
    - Includes error handling for Redis connection issues and invalid job formats. Ensures DB/Redis connections are active before processing.

 

## 🎨 Frontend Components

### 1. Main Application (`main.js`, `App.vue`, `index.html`)

- `main.js`: Entry point for the Vue application. Creates the Vue app instance, mounts the root component (`App.vue`), and installs the Vue Router plugin. Imports global CSS (`index.css`).
- `App.vue`: The root Vue component. Primarily renders the `<router-view>` which displays the component corresponding to the current route.
- `index.html`: The main HTML file. Includes the root div (`<div id="app">`) where the Vue app is mounted and links the `main.js` script.

### 2. Router (`src/router/index.js`)

Purpose: Defines application routes and navigation guards.
Key Features:
- History Mode: Uses `createWebHistory` for clean URLs.
- Routes: Defines paths for login, registration, and main application pages (Dashboard, Invoices, Vendors, Purchase Orders, PO Editor, PO View) nested under `AppLayout`. Includes a catch-all 404 route.
- Authentication Guard (`beforeEach`):
    - Checks `meta: { requiresAuth: true }` on routes.
    - Checks for user data (token) in `localStorage`.
    - Redirects unauthenticated users trying to access protected routes to `/login`.
    - Redirects authenticated users trying to access `/login` or `/register` to the Dashboard (`/`).
    - Updates the document title based on route metadata.
- Props: Passes route params (`:id`) as props to `POEditor` and `POView` components.

### 3. Layout (`src/layouts/AppLayout.vue`)

Purpose: Provides the main application structure for authenticated users, including the sidebar and header.
Components:
- `SideNavbar.vue`: Persistent navigation sidebar.
- `AppHeader.vue`: Top header bar.
- `<router-view>`: Renders the content of the current authenticated route within the main content area.
- Print Styles: Includes `@media print` styles to hide the navbar/header and optimize the main content area for printing.

### 4. Common Components (`src/components/common/`)

- `SideNavbar.vue`: Displays navigation links (Dashboard, Invoices, POs, Vendors) using `router-link`. Highlights the active link. Hidden on mobile (md breakpoint) and during printing.
- `AppHeader.vue`: Displays a welcome message with the user's name (fetched from `localStorage` via `authService`) and a Logout button. Hidden during printing.
- PO Templates (`po-templates/`): Vue components representing different PO visual layouts (`ProfessionalTemplate.vue`, `SimpleTemplate.vue`, `ModernTemplate.vue`, `StripeTemplate.vue`).
    - Receive PO data via `v-model` (using `modelValue` prop and `update:modelValue` emit).
    - Provide input fields for editing PO details (addresses, items, notes, terms, etc.).
    - Implement internal calculations (item totals, subtotals, taxes, grand total) using computed properties and watchers.
    - Update the `modelValue` (the main PO object) with calculated values.
    - Emit a `saveAndContinue` event when the user clicks the save button.
    - Some templates include logo upload functionality (reading the file as Base64).

### 5. Pages (`src/pages/`)

- `Login.vue`: Provides email/password inputs and handles login attempts using `authService.login`. Displays errors and loading states. Redirects to Dashboard on success.
- `Register.vue`: Provides name/email/password inputs and handles registration using `authService.register`. Displays messages (success/error) and loading states. Redirects to Login on success.
- `Dashboard.vue`: Basic welcome page with quick links to other sections. Includes commented-out example code for fetching and displaying stats.
- `Invoices.vue`:
    - Displays a list of invoices fetched via `invoiceService.getAllInvoices`.
    - Shows invoice details (extracted PO number, vendor name, amount, status).
    - Provides a file upload button (`<input type="file">`) triggering `handleFileUpload` which calls `invoiceService.uploadInvoice`. Displays upload progress/errors.
    - Implements polling (`setInterval`) to automatically refresh the invoice list when an invoice has 'Processing' status.
    - Shows actions based on invoice status: 'Generate PO' (opens template modal), 'Edit PO' (navigates to editor), 'Approve', 'Delete'. Displays 'Processing...' or 'Error' indicators.
    - Includes a modal (`showTemplateModal`) for selecting a PO template (`templateOptions` imported from config) before generating a PO. Calls `purchaseOrderService.createPO` and navigates to the editor on success.
- `Vendors.vue`: Displays a list of vendors fetched via `vendorService.getAllVendors`. Provides 'Add New Vendor' button which opens a modal (`showAddModal`) for creating vendors via `vendorService.createVendor`. Includes a 'Delete' action calling `vendorService.deleteVendor`.
- `PurchaseOrders.vue`: Displays a list of POs fetched via `purchaseOrderService.getAllPurchaseOrders`. Shows PO number, vendor, date, amount, status. Provides actions: 'View', 'Edit' (navigate to respective pages), 'Delete' (calls `purchaseOrderService.deletePO`).
- `POEditor.vue`:
    - Fetches PO data based on route parameter `:id` using `purchaseOrderService.getPOById`. Handles loading and error states.
    - Includes robust initialization logic to ensure nested objects/arrays (like `orderBy`, `items`) exist in the fetched PO data to prevent template errors. Parses/defaults item quantities/rates.
    - Provides a dropdown to switch the `po.templateName`.
    - Dynamically loads and renders the appropriate PO template component (from `po-templates/`) based on `po.templateName` using `defineAsyncComponent` and `shallowRef`. Passes the `po` object to the template using `v-model`.
    - Listens for the `saveAndContinue` event from the template component and calls `purchaseOrderService.updatePO` to save the modified `po` object. Navigates to the PO View page on successful save.
- `POView.vue`:
    - Fetches PO data based on route parameter `:id` using `purchaseOrderService.getPOById`. Handles loading and error states.
    - Displays a read-only view of the PO details (header, logo, dates, addresses, items table, totals, notes, terms).
    - Provides action buttons: 'Print / Save as PDF' (triggers browser print `window.print()`), 'Download as PDF (Basic)', 'Download as Word (Basic)', 'Download as PNG', 'Back to Edit'.
    - PDF/Word download links point to the backend generation routes (`/api/generate/...`) and include the auth token as a query parameter.
    - PNG download uses `html2canvas` to capture the PO display area (`poViewRef`) and initiates a download.
    - Includes specific print styles (`@media print`) to optimize the layout for printing.
- `NotFound.vue`: Simple 404 page displayed by the catch-all route.

### 6. API Services (`src/services/`)

Purpose: Encapsulate API communication logic using Axios.

- `apiClient.js`: Creates and configures the base Axios instance.
    - Sets `baseURL` based on `VITE_API_BASE_URL` environment variable or a default.
    - Includes a request interceptor to automatically add the JWT token (`x-auth-token`) from `localStorage` to outgoing requests.
    - Includes a response interceptor to handle 401 Unauthorized errors by clearing `localStorage` and redirecting to the login page.
- `authService.js`: Functions for `login`, `register`, `logout` (clears `localStorage`), and `getCurrentUser` (retrieves user data from `localStorage`). Uses `apiClient` for requests. Stores user data (including token) in `localStorage` on successful login.
- `invoiceService.js`: Functions for `uploadInvoice` (sends `FormData` with correct headers), `getAllInvoices`, `approveInvoice`, `deleteInvoice`. Uses `apiClient`.
- `vendorService.js`: Functions for `getAllVendors`, `createVendor`, `deleteVendor` (includes commented-out `getVendorById`, `updateVendor`). Uses `apiClient`.
- `purchaseOrderService.js`: Functions for `getAllPurchaseOrders`, `getPOById`, `createPO`, `updatePO`, `deletePO`. Uses `apiClient`.

### 7. Configuration (`src/config/poTemplates.js`)

Purpose: Defines the list of available PO template names used in the `POEditor` dropdown and for dynamic component loading.

 

## 📊 Data Flow & API Endpoints

### Primary User Flows

1.  Invoice Upload & Processing:
    - User selects file (Image/PDF) in `Invoices.vue`.
    - `handleFileUpload` triggers `POST /api/invoices/upload` with file data.
    - `invoiceController.uploadInvoice` saves file to `/uploads`, creates `Invoice` doc (status: Processing), pushes job `{invoiceId}` to Redis queue `invoice_processing_ppx`.
    - `Invoices.vue` polls `GET /api/invoices`.
    - `ppx-ocr-service` (`main.py`) pops job from Redis.
    - `process_invoice` fetches Invoice doc, reads file from shared volume.
    - `call_perplexity_ocr` sends file (Base64) + prompt to Perplexity API.
    - Service parses response, calls `update_invoice_in_db` to update Invoice doc (status: Pending/Error, adds `extractedData`).
    - `Invoices.vue` polling reflects updated status.

2.  PO Generation & Editing:
    - User clicks 'Generate PO' on a 'Pending' invoice in `Invoices.vue`.
    - Template selection modal appears.
    - User selects template, clicks 'Generate PO'.
    - `handleGeneratePO` calls `POST /api/purchase-orders` with `{ invoiceId, templateName }`.
    - `purchaseOrderController.createPO` fetches Invoice, uses `extractedData` to create new `PurchaseOrder` doc (status: Draft), links it to the Invoice.
    - Frontend redirects to `POEditor.vue` (`/purchase-orders/:newPOId/edit`).
    - `POEditor.vue` fetches the new PO data.
    - User edits details using the selected template component. Template calculates totals.
    - User clicks 'Save & Continue'.
    - `saveAndContinue` calls `PUT /api/purchase-orders/:poId` with the full updated PO object.
    - `purchaseOrderController.updatePO` saves changes.
    - Frontend redirects to `POView.vue` (`/purchase-orders/:poId/view`).

3.  PO Viewing & Downloading:
    - User navigates to `POView.vue`.
    - Component fetches PO data via `GET /api/purchase-orders/:poId`.
    - Data is displayed read-only.
    - User clicks 'Download PDF' -> link triggers `GET /api/generate/pdf/:poId?token=...`.
    - User clicks 'Download Word' -> link triggers `GET /api/generate/word/:poId?token=...`.
    - `fileGenerationController` generates and streams the file.
    - User clicks 'Download PNG' -> `html2canvas` captures the view, initiates browser download.
    - User clicks 'Print' -> `window.print()` uses print-specific CSS.

### API Endpoints

#### Authentication (`/api/auth`)

- `POST /register`: Creates a new user account.
    - Request: `{ name, email, password }`
    - Response: `{ token }` or error
- `POST /login`: Authenticates a user.
    - Request: `{ email, password }`
    - Response: `{ token, user: { id, name, email } }` or error

#### Invoices (`/api/invoices`) - Requires Auth

- `POST /upload`: Uploads an invoice file for processing.
    - Request: `multipart/form-data` with field `invoiceFile` containing the file (Image/PDF, max 10MB).
    - Response: The newly created `Invoice` object (status: Processing) or error.
- `GET /`: Retrieves all invoices.
    - Response: Array of `Invoice` objects.
- `PUT /:id/approve`: Sets an invoice status to 'Approved'.
    - Response: Updated `Invoice` object or error.
- `DELETE /:id`: Deletes an invoice and its associated file.
    - Response: `{ msg: 'Invoice removed successfully' }` or error (e.g., if linked to PO).

#### Vendors (`/api/vendors`) - Requires Auth

- `POST /`: Creates a new vendor.
    - Request: `{ name, email?, phone?, gstin? }`
    - Response: The created `Vendor` object or error.
- `GET /`: Retrieves all vendors.
    - Response: Array of `Vendor` objects.
- `GET /:id`: Retrieves a specific vendor by ID.
    - Response: The `Vendor` object or 404 error.
- `PUT /:id`: Updates a vendor.
    - Request: `{ name, email?, phone?, gstin? }`
    - Response: The updated `Vendor` object or error.
- `DELETE /:id`: Deletes a vendor.
    - Response: `{ msg: 'Vendor removed successfully' }` or error.

#### Purchase Orders (`/api/purchase-orders`) - Requires Auth

- `POST /`: Creates a new Purchase Order from a processed Invoice.
    - Request: `{ invoiceId, templateName }`
    - Response: The created `PurchaseOrder` object or error.
- `GET /`: Retrieves all Purchase Orders.
    - Response: Array of `PurchaseOrder` objects (with populated invoice details).
- `GET /:id`: Retrieves a specific Purchase Order by ID.
    - Response: The `PurchaseOrder` object (with populated invoice details) or 404 error.
- `PUT /:id`: Updates a Purchase Order.
    - Request: The full updated `PurchaseOrder` object (excluding `_id`, `invoice`).
    - Response: The updated `PurchaseOrder` object or error.
- `DELETE /:id`: Deletes a Purchase Order.
    - Response: `{ msg: 'Purchase Order removed successfully' }` or error.

#### File Generation (`/api/generate`) - Requires Auth (Header or Query Token)

- `GET /pdf/:id`: Generates and downloads a PDF for the specified PO ID.
    - Response: `application/pdf` file stream.
- `GET /word/:id`: Generates and downloads a Word (.docx) file for the specified PO ID.
    - Response: `application/vnd.openxmlformats-officedocument.wordprocessingml.document` file stream.

#### Health Check (`/api/health`)

- `GET /health`: Basic health check endpoint.
    - Response: String `Backend Server is running`.

 

## 🧠 Component Logic Explanation

### Key Backend Logic

- Invoice Processing Trigger (`invoiceController.js`): Upon successful file upload via `POST /api/invoices/upload`, the controller saves the file, creates an `Invoice` database entry with `status: 'Processing'`, and then pushes a JSON message containing only the `invoiceId` onto a Redis list named `REDIS_QUEUE_NAME` using `LPUSH`. This decouples the upload process from the potentially long-running OCR task.
- OCR Job Consumption (`main.py`): The Python worker loop continuously attempts to pop a job from the `REDIS_QUEUE_NAME` list using `BRPOP` (blocking pop). When a job is received, it extracts the `invoiceId` and calls `process_invoice`.
- PO Creation from Extraction (`purchaseOrderController.js`): The `createPO` function is triggered by `POST /api/purchase-orders`. It fetches the specified `Invoice` by `invoiceId`, verifies its status is 'Pending' or 'Approved' (not 'Processing' or 'Error'), and then meticulously maps fields from the nested `invoice.extractedData` object (which mirrors the Perplexity JSON output) to the `PurchaseOrder` schema structure. This involves:
    - Parsing dates using `parseFlexibleDate` (handling formats like DD/MM/YY).
    - Parsing quantities and rates using `parseSafeFloat` (handling currency symbols, commas, non-numeric text like "kilos").
    - Calculating initial item amounts, taxes (CGST/SGST based on `gstRate`), and item totals.
    - Calculating overall subtotal, tax totals, and grand total based on the mapped items and extracted discount.
    - Setting default status to 'Draft' and linking the PO back to the invoice.
- PDF/Word Generation (`fileGenerationController.js`): Both `generatePDF` and `generateWord` fetch the complete `PurchaseOrder` data by ID. They then use their respective libraries (`pdfkit`, `docx`) to programmatically construct the document element by element (text, tables, images, formatting) based on the PO data. The final document is streamed/sent directly in the HTTP response with appropriate `Content-Type` and `Content-Disposition` headers.

### Key Frontend Logic

- Invoice Upload (`Invoices.vue`): Uses a hidden file input triggered by a styled label. The `change` event handler grabs the selected file, creates `FormData`, appends the file under the key `invoiceFile`, sets an `uploading` state, and calls `invoiceService.uploadInvoice`. Success refreshes the invoice list; failure displays an error message.
- Processing State & Polling (`Invoices.vue`): A computed property `isProcessingAny` checks if any invoice in the list has `status === 'Processing'`. When `fetchInvoices` runs, if `isProcessingAny` is true and polling isn't active, `setInterval` is started to call `fetchInvoices` every 5 seconds. If `isProcessingAny` becomes false, `clearInterval` stops the polling. An `onUnmounted` hook also ensures polling stops when leaving the page.
- PO Generation Trigger (`Invoices.vue`): The 'Generate PO' button (visible only for 'Pending' invoices without a linked PO) calls `openTemplateModal`, passing the specific invoice data. This sets `currentInvoice` and shows the modal. Inside the modal, selecting a template updates `selectedTemplate`. Clicking the final 'Generate PO' button calls `handleGeneratePO`, which sends the `currentInvoice._id` and `selectedTemplate` to `purchaseOrderService.createPO`. On success, it fetches updated POs, closes the modal, and navigates to the `POEditor` page for the new PO ID.
- Dynamic Template Loading (`POEditor.vue`): The editor uses a computed property `currentTemplateComponent`. This property checks the `po.templateName` value. Based on this name, it returns the corresponding asynchronously loaded component from the `templates` object (which uses `defineAsyncComponent` and `shallowRef` for efficiency). The `<component :is="currentTemplateComponent">` tag dynamically renders the chosen template. A default template (Professional) is used if the `templateName` is invalid or missing.
- PO Editing & Saving (`POEditor.vue`, Template Components): The `POEditor` passes the fetched (and initialized) `po` data object down to the dynamic template component using `v-model="po"`. The template component uses this `modelValue` prop and emits `update:modelValue` when changes occur (either direct inputs or internal calculations via watchers/computed properties). When the user clicks 'Save & Continue' in the template, it ensures all final calculations are updated on the `po` object and then emits `saveAndContinue`. The `POEditor` listens for this event and calls `purchaseOrderService.updatePO` with the potentially modified `po` object.
- PO Downloads (`POView.vue`):
    - PDF/Word: Computed properties `pdfUrl` and `wordUrl` construct the backend download URLs (`/api/generate/.../:id`), appending the auth token from `localStorage` as a query parameter (`?token=...`). These URLs are used in standard `<a>` tag `href` attributes.
    - PNG: The `downloadPNG` function uses `html2canvas` to render the DOM element referenced by `poViewRef` onto a canvas. It gets the canvas data URL (`toDataURL('image/png')`), creates a temporary link element (`<a>`), sets its `href` to the data URL and `download` attribute to a filename, programmatically clicks the link, and then removes it.

 

## 🤖 AI Integration (Perplexity OCR)

The core AI functionality is handled by the dedicated Python OCR service (`ppx-ocr-service`).

Process:
1.  Trigger: The backend queues a job in Redis containing only the `invoiceId` when an invoice file is uploaded.
2.  Job Pickup: The Python service picks up the `invoiceId` from the Redis queue.
3.  File Access: The service retrieves the `filePath` from the Invoice document in MongoDB and reads the corresponding file (e.g., `/app/uploads/invoiceFile-....jpg`) from the shared Docker volume.
4.  API Call (`call_perplexity_ocr`):
    * The file content is read and Base64 encoded.
    * A specific prompt is constructed. This prompt explicitly asks the Perplexity model (`sonar`) to act as an OCR/data extraction expert and to return the extracted information strictly in a predefined JSON format. It lists all required fields and sub-fields (poNumber, poDate, discount, notes, terms, orderBy, orderTo, shippingDetails, items list).
    * The prompt text and the Base64 encoded image data (formatted as a data URL, e.g., `data:image/jpeg;base64,...`) are sent as part of the `messages` payload to the Perplexity Chat Completions API endpoint (`https://api.perplexity.ai/chat/completions`).
    * The `max_tokens` parameter is set to 1024 to limit the length of the generated response.
5.  Response Handling:
    * The service receives the JSON response from Perplexity.
    * It attempts to locate the actual JSON string containing the extracted data, often nested within the API response structure (e.g., inside `choices[0].message.content`).
    * It cleans the string (finding the first `{` and last `}`) and parses it into a Python dictionary using `json.loads`.
    * Error handling is included for API errors, network issues, timeouts, file not found, and JSON parsing failures.
6.  Database Update: The successfully parsed dictionary (or an error message) is saved to the `extractedData` field of the corresponding Invoice document in MongoDB, and the invoice `status` is updated to 'Pending' or 'Error'.

Privacy Considerations:
-   No PII Sent Directly (Except Image): The service does not explicitly send user account information or other application data to the Perplexity API. However, the invoice image itself is sent, which often contains sensitive business information (names, addresses, item details, pricing).
-   Prompt Context: The prompt itself is generic and provides the desired JSON structure but does not include user-specific data beyond the file's MIME type.
-   Data Storage: The extracted data, which may contain sensitive details from the invoice, is stored within the `Invoice` document in the MongoDB database.

 

## 🚀 Local Development Setup

### Prerequisites

-   Docker and Docker Compose: Essential for running the multi-container setup. Get them from [Docker Desktop](https://www.docker.com/products/docker-desktop/).
-   Node.js & npm: Required by the root `package.json` scripts for interacting with Docker Compose, and potentially for direct frontend/backend development if not using Docker exclusively. Version 18+ recommended based on Dockerfiles.
-   Python: Required by the OCR service. Version 3.9+ recommended based on Dockerfile.
-   Git: For cloning the repository.
-   Text Editor/IDE: VS Code, WebStorm, PyCharm, etc.

### Environment Configuration

1.  Clone Repository: `git clone <your-repo-url> ppx-po-gen && cd ppx-po-gen`
2.  Create `.env` file: Copy `.env.example` (if provided) or create a `.env` file in the project root.
3.  Fill `.env` Variables:
    * `MONGO_URI`: Your MongoDB connection string (e.g., from MongoDB Atlas). Ensure the database name is included.
    * `JWT_SECRET`: A strong, unique secret key for signing authentication tokens. Change the default value.
    * `PERPLEXITY_API_KEY`: Your API key obtained from Perplexity AI.
    * `PERPLEXITY_API_ENDPOINT`: Set to `https://api.perplexity.ai/chat/completions`.
    * `PORT`, `REDIS_HOST`, `REDIS_PORT`, `REDIS_QUEUE_NAME`: Usually keep the defaults (`5002`, `redis`, `6379`, `invoice_processing_ppx`) as they are configured for Docker Compose internal networking.
    * `VITE_API_BASE_URL`: For frontend development outside Docker, set this to the backend URL (e.g., `http://localhost:5002/api`). The `docker-compose.yml` overrides this for containerized frontend.

### Running the Application (Docker Compose)

1.  Build and Start Containers:
    ```bash
    npm start
    # or directly:
    # docker-compose up --build -d # -d runs in detached mode
    ```
    This command reads the `docker-compose.yml`, builds the images for `frontend`, `backend`, and `ppx-ocr-service` if they don't exist or have changed, and starts all defined services (`frontend`, `backend`, `ppx-ocr-service`, `redis`) including creating necessary volumes (`uploads-data`, `redis-data`).

2.  Access Services:
    * Frontend: `http://localhost:8081` (Nginx container proxies to Vue app)
    * Backend API: `http://localhost:5002` (Direct access to Node.js app)
    * Redis: Accessible internally to other containers on `redis:6379`. Exposed externally on `localhost:6379` if needed for debugging.

3.  View Logs:
    ```bash
    npm run logs
    # or directly:
    # docker-compose logs -f # -f follows logs
    # docker-compose logs -f ppx-ocr-service # Logs for a specific service
    ```

4.  Stop Containers:
    ```bash
    npm stop
    # or directly:
    # docker-compose down # Stops and removes containers, networks
    # docker-compose down -v # Stops and removes containers, networks, AND volumes
    ```

### Running Services Individually (Optional, for focused development)

-   Backend: Navigate to `backend/`, set up `.env` (adjust `REDIS_HOST` to `localhost` if running Redis locally or via `docker run`), run `npm install`, then `npm run dev` (uses nodemon). Requires MongoDB and Redis running separately.
-   Frontend: Navigate to `frontend/`, set up `.env` (ensure `VITE_API_BASE_URL` points to backend), run `npm install`, then `npm run dev` (uses Vite dev server).
-   OCR Service: Navigate to `ppx-ocr-service/`, set up `.env` (adjust `MONGO_URI` and `REDIS_HOST` if needed), set up Python environment (`python -m venv venv`, `source venv/bin/activate`, `pip install -r requirements.txt`), run `python main.py`. Requires MongoDB and Redis running separately.

 

## 🧪 Testing & Deployment

### Testing Strategy

-   Backend: (No explicit test files provided in upload). Suggested approach:
    -   Unit Tests: Use a framework like Jest or Mocha/Chai to test individual controller functions, helper functions (`parseSafeFloat`, `parseFlexibleDate`), and potentially model validation logic. Mock database interactions and external API calls.
    -   Integration Tests: Use Supertest to test API endpoints, ensuring routes, middleware (auth), controllers, and database interactions work together correctly. Test CRUD operations, file upload, PO generation logic.
-   Frontend: (No explicit test files provided in upload). Suggested approach:
    -   Unit Tests: Use Vitest or Jest with Vue Testing Library to test individual components (e.g., form validation, template calculations, state changes). Mock API service calls.
    -   End-to-End Tests: Use Cypress or Playwright to simulate user flows (login, upload invoice, generate PO, edit PO, download PO).
-   OCR Service: (No explicit test files provided in upload). Suggested approach:
    -   Unit Tests: Use `pytest` to test helper functions (`connect_*`, `ensure_connections`, JSON parsing logic within `call_perplexity_ocr`, `update_invoice_in_db`). Mock external dependencies (`redis`, `pymongo`, `requests`).
    -   Integration Tests: Test the `process_invoice` function with mock Redis jobs and a test MongoDB instance, potentially mocking the Perplexity API call or using a dedicated test key with caution.

### Deployment

-   Primary Method: Docker Compose is suitable for single-server deployments.
    1.  Ensure Docker and Docker Compose are installed on the target server.
    2.  Copy the entire project structure or use Git to clone it.
    3.  Create a production `.env` file on the server with appropriate `MONGO_URI`, a strong `JWT_SECRET`, the `PERPLEXITY_API_KEY`, etc. Ensure `VITE_API_BASE_URL` in `docker-compose.yml` correctly points to the backend service name if needed, or is handled by Nginx proxy.
    4.  Run `docker-compose up --build -d` to build and start all services in detached mode.
-   Alternative Methods:
    -   Deploy containers individually to platforms like Kubernetes, AWS ECS, Google Cloud Run. This requires more complex configuration for networking, volumes, and environment variables.
    -   Deploy backend/frontend separately (e.g., backend to a PaaS like Heroku/Render, frontend to Vercel/Netlify). Requires careful CORS configuration and setting `VITE_API_BASE_URL` during the frontend build process. The OCR service would likely still run as a separate container or process connected to Redis/MongoDB.
-   Nginx Configuration (`frontend/nginx.conf`): The frontend Docker service uses Nginx. The provided config serves the built Vue app (`/usr/share/nginx/html`) and proxies requests starting with `/api/` to the backend service (`http://backend:5002`). It includes `try_files` for Vue Router's history mode.

### Performance Considerations

-   Backend:
    -   Database indexing is used on key fields (`User.email`, `Invoice.invoiceNumber`, `Invoice.status`, `Invoice.createdAt`, `Vendor.name`, `PurchaseOrder.poNumber`, `PurchaseOrder.status`, `PurchaseOrder.invoice`).
    -   File uploads are handled via streaming by Multer to disk, preventing large files from consuming excessive memory.
    -   PDF/Word generation can be CPU-intensive; consider background jobs for very large/complex POs if performance becomes an issue.
-   OCR Service:
    -   The Perplexity API call is the main bottleneck; timeouts are set (120s).
    -   The service is designed as a background worker, not blocking the main user flow. Multiple worker instances could be run for higher throughput if needed (requires coordination or different queueing strategy).
-   Frontend:
    -   Uses Vite for optimized builds.
    -   Dynamically imports PO template components (`defineAsyncComponent`) to reduce initial bundle size.
    -   Polling on the Invoices page could be optimized (e.g., using WebSockets or Server-Sent Events) but is simple to implement.

### Security Checklist

-   ✅ Authentication: JWT is used to protect backend routes. `authMiddleware` verifies tokens. `authFromQueryOrHeader` adds flexibility for download links.
-   ✅ Password Hashing: `bcryptjs` is used to hash user passwords before storing.
-   ✅ Input Validation: Mongoose schemas provide basic type validation on save/update. Controller logic adds some checks (e.g., required fields, ID validity). File uploads have type and size limits.
-   ✅ Authorization: Currently implicit (any logged-in user can access all data). Consider adding role-based or owner-based access control if needed.
-   ✅ Environment Variables: Sensitive keys (`JWT_SECRET`, `MONGO_URI`, `PERPLEXITY_API_KEY`) are stored in `.env` and not committed to Git (per `.gitignore`).
-   ✅ CORS: Enabled via `cors` middleware, allowing frontend access. For production, configure `ALLOWED_ORIGINS` restrictively.
-   ✅ File Uploads: Multer saves files with unique names to prevent collisions/overwrites. Path traversal isn't an obvious risk as paths are generated internally. Ensure the `/app/uploads` directory isn't directly accessible via HTTP.
-   ⚠️ Rate Limiting: Not explicitly implemented; consider adding middleware (e.g., `express-rate-limit`) to backend routes in production.
-   ⚠️ Dependency Security: Regularly audit dependencies (npm audit, pip audit) for known vulnerabilities.

### Monitoring & Logging

-   Backend: Uses `console.log` and `console.error` for logging various events (server start, DB connection, API calls, errors). Consider using a structured logger (like Winston or Pino) for better log management and analysis in production.
-   OCR Service: Uses Python's built-in `logging` module configured for INFO level, logging connection attempts, job reception, API calls, status updates, and errors.
-   Frontend: Uses `console.log` and `console.error` for debugging. Consider integrating a frontend error tracking service (e.g., Sentry) for production.

 

## 📝 Conclusion

The PPX PO Gen system provides a streamlined workflow for automating Accounts Payable tasks, specifically focused on converting invoices into Purchase Orders.

### Key Strengths

-   Automated Data Extraction: Leverages the Perplexity API via a dedicated microservice to extract structured data from invoice documents (images/PDFs), reducing manual data entry.
-   Workflow Automation: Guides users from invoice upload through processing, PO generation, editing, and finalization.
-   Flexibility: Offers multiple PO templates for editing and viewing, allowing customization. Provides downloads in PDF, Word, and PNG formats.
-   Modern Tech Stack: Utilizes current technologies (Vue 3, Node.js, FastAPI, MongoDB, Redis, Docker) for a robust and maintainable application.
-   Containerized: Docker Compose setup simplifies development, deployment, and dependency management.
-   Asynchronous Processing: Uses Redis as a queue to decouple the time-consuming OCR process from the main user interaction flow, improving responsiveness.

### Potential Areas for Enhancement

-   Advanced Vendor Matching: Automatically link `orderTo` details to existing `Vendor` records.
-   UI/UX Refinements: Improve loading indicators, provide more granular feedback during processing, enhance template previews.
-   Error Handling Robustness: More specific error messages on the frontend, potentially more sophisticated retry logic in the OCR service.
-   Testing Coverage: Implement comprehensive unit, integration, and end-to-end tests.
-   Security Hardening: Add rate limiting, more granular authorization, and dependency scanning.
-   Scalability: Consider strategies for scaling the OCR service (multiple workers) and backend if handling very high volume.
-   Observability: Implement structured logging and monitoring across all services.

The system provides a solid foundation for AP automation, significantly reducing manual effort in PO creation by leveraging AI for initial data capture.

 
Documentation generated based on provided source code files.
 
