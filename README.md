# TechNova AI Store

A BCA academic project for shopping electronics and comparing products with a requirement-based recommendation engine. Recommendations are calculated from the current catalog data; the assistant does not call an external LLM or return random products.

## Project Structure

```text
MINI TECH STORE/
|-- manage.py
|-- requirements.txt
|-- db.sqlite3
|-- config/                 Django settings, root URLs, WSGI
|-- accounts/               Registration, login, profile
|-- products/               Product, category, brand, catalog, seed command
|-- cart/                   Cart items and persistent wishlist
|-- orders/                 Checkout, order history, order admin
|-- recommendations/        Requirement parser, scorer, comparison, history
|-- chatbot/                JSON endpoint for requirement extraction
|-- dashboard/              Staff-only sales and catalog summary
|-- templates/              Shared layout and app pages
|-- static/css/style.css    Responsive storefront styling
|-- static/js/app.js        Small client-side enhancements
|-- media/                  Uploaded media location
```

Core product specifications are typed fields on `products.Product` (RAM, storage, processor, GPU, display, battery, and usage scores). Optional category-specific attributes live in the related `products.ProductSpecification.details` JSON field and can be edited inline in Django admin. The database is SQLite for development; the database engine is isolated in `config/settings.py` for later PostgreSQL or MySQL configuration.

## Install And Run (Windows PowerShell)

From the project directory:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python manage.py migrate
python manage.py seed_data
python manage.py createsuperuser
python manage.py runserver
```

Open `http://127.0.0.1:8000/`. The Django admin is at `http://127.0.0.1:8000/admin/`; the staff dashboard is at `/dashboard/`.

`seed_data` creates or updates the small sample catalog, including sale prices, so the storefront and advisor can be tried immediately. Sample product images are loaded from Unsplash URLs and need an internet connection; product image URLs can be changed in Django admin.

## Main Modules

- `products/models.py`, `products/views.py`, `products/admin.py`, and `products/management/commands/seed_data.py`: catalog and specification models, listing/search filters, product detail, admin controls, and sample products.
- `recommendations/engine.py`: natural-language extraction and deterministic recommendation scoring.
- `recommendations/views.py` and `templates/recommendations/advisor.html`: structured/natural-language advisor form, results, comparisons, and signed-in recommendation history.
- `cart/models.py` and `cart/views.py`: user cart, stock-limited quantity updates, and persistent wishlist.
- `orders/models.py`, `orders/views.py`, and `orders/admin.py`: checkout, order history, and staff order management.
- `accounts/`: Django user registration, authentication, and profile pages.
- `dashboard/`: staff-only catalog, user, order, and sales totals.

## Recommendation Method

The advisor extracts category, budget, RAM, storage, usage, GPU, processor, brand, display, and battery requirements. Structured form values take precedence over parsed values. Products that are out of stock or outside the requested category are excluded.

The compatibility score is a weighted sum, clamped to a 0–100 scale:

- Budget fit: 25%
- Usage/performance fit: 25%
- Explicit requirement fit: 25%
- Cosine similarity across requested specification features: 15%
- Catalog rating: 10%

Each result includes reasons for satisfied and unmet constraints. Requirement extraction is local rule-based Python; no API key is needed. The parser and scoring logic are modular so an LLM or a richer model can be added later without exposing credentials in the browser.

## Try The Advisor

Open `/advisor/` and enter:

> I need a laptop under ₹70,000 for programming, gaming and AI/ML. I want at least 16GB RAM and 512GB SSD with a dedicated GPU.

The results should rank products using their saved specifications and show the score breakdown reasons. Sign in first if you want the recommendation to appear later as the product's compatibility score.

## Migrations And Tests

Apply the included schema migrations after installing dependencies:

```powershell
python manage.py migrate
```

When changing models, generate and apply a migration:

```powershell
python manage.py makemigrations
python manage.py migrate
```

Run the automated tests and configuration checks:

```powershell
python manage.py test
python manage.py check
```

The tests cover natural-language extraction, recommendation ordering/explanations, signed-in recommendation persistence, cart quantity limits, and wishlist persistence.

## Razorpay Demo Checkout

Checkout supports COD, Razorpay test mode, and Razorpay live mode. For local testing, run `Copy-Item .env.example .env` in PowerShell and enter your **Test Mode** Key ID and Key Secret. Only `rzp_test_` keys enable test mode.

For real payments, deploy the site on an HTTPS domain first. Set `RAZORPAY_MODE=live`, `DEBUG=False`, `SECURE_SSL_REDIRECT=True`, the production `ALLOWED_HOSTS`, and your merchant **Live Mode** Key ID (`rzp_live_...`) and Key Secret in the server environment. Live mode is deliberately disabled unless debug mode is off and HTTPS enforcement is enabled. Obtain live keys through your verified Razorpay merchant account; never put them in source control or browser code. Restart the service after changing environment settings.

The application creates an INR order through Razorpay's Orders API, verifies the returned payment signature server-side, and only then marks the order paid, deducts stock, and updates the cart. Each order records whether it used test or live mode. Failed or dismissed payments do not consume stock. Tests mock the gateway API, so running the test suite never contacts Razorpay or charges money. Live payment processing still requires valid merchant credentials and an internet-accessible HTTPS deployment; these cannot be supplied by the codebase itself.
