# Gravir.pro — Technical Plan

## Only What Matters

Build an online store for personalized wooden gifts. The simpler, the better.

---

## What We Need

### Must Have
1. Products with photos and prices
2. Cart
3. Checkout (name, phone, address)
4. Payment (bank transfer)
5. Email confirmation

### What We Don't Need Now
- User accounts
- Reviews
- Order tracking
- Multi-language
- Complex design tool

---

## Pages

```
/                  → Home
/products          → Products
/product/{slug}   → Product detail
/cart              → Cart
/checkout          → Checkout
/thank-you         → Thank you
/admin             → Admin
/admin/orders      → Orders
```

---

## Database

```
users              → Name, email, phone, is_admin
categories        → Name, description, image
products          → Name, description, price, image, category_id
orders            → user_id, name, phone, address, total, status, notes
order_items       → order_id, product_id, quantity, price
```

---

## Technologies

```bash
# Installation
composer create-project laravel/laravel gravir
cd gravir
composer require livewire/livewire
npm install
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Packages
- **Laravel 12** — Web framework
- **Livewire 4** — Interactivity without JavaScript
- **Tailwind CSS 4** — Styling
- **Stripe** — Payments (later)
- **Intervention Image** — Image processing

---

## Structure

```
app/
├── Http/
│   ├── Controllers/
│   │   ├── HomeController.php
│   │   ├── ProductController.php
│   │   └── CheckoutController.php
│   └── Livewire/
│       ├── Cart.php
│       └── Checkout.php
├── Models/
│   ├── Product.php
│   ├── Order.php
│   └── OrderItem.php
```

---

## Step by Step

### Week 1: Foundation
- [ ] Laravel project
- [ ] Database migrations
- [ ] Models (Product, Order)
- [ ] Home page

### Week 2: Products
- [ ] Product listing
- [ ] Product page
- [ ] Image upload

### Week 3: Cart + Checkout
- [ ] Cart (Livewire)
- [ ] Checkout form
- [ ] Save to database

### Week 4: Admin + Launch
- [ ] Order listing
- [ ] Status update
- [ ] Email confirmation
- [ ] **LAUNCH**

---

## Code Examples

### Product Model
```php
class Product extends Model
{
    protected $fillable = ['name', 'description', 'price', 'image', 'category_id'];
}
```

### Checkout Livewire Component
```php
class Checkout extends Component
{
    public $name = '';
    public $phone = '';
    public $address = '';
    public $notes = '';
    
    protected $rules = [
        'name' => 'required|min:2',
        'phone' => 'required|min:10',
        'address' => 'required',
    ];
    
    public function submit()
    {
        $this->validate();
        
        $order = Order::create([
            'name' => $this->name,
            'phone' => $this->phone,
            'address' => $this->address,
            'notes' => $this->notes,
            'total' => session()->get('cart_total', 0),
            'status' => 'new'
        ]);
        
        // Save cart items...
        // Send email...
        
        session()->forget('cart');
        
        return redirect('/thank-you');
    }
    
    public function render()
    {
        return view('livewire.checkout');
    }
}
```

---

## Payment

### Option 1: Bank Transfer (Now)
- Customer sees bank details
- Pays via bank
- We see confirmation and ship

### Option 2: Stripe (Later)
- Card payments
- Auto confirmation

---

## Design

### Principles
- Lots of white space
- One great photo per product
- Clear buttons
- Fast loading
- Mobile-first

### Colors
- Primary: Wood (brown/beige)
- Accent: Green (action)
- Text: Dark gray

---

## Photos

Photos are everything. Every product has one great photo.

- Background: White
- Light: Natural
- Angle: Straight on
- Resolution: 1200x1200

---

## Admin

Just an order list:

```
| # | Customer | Products | Status | Action        |
|---|----------|----------|--------|---------------|
| 1 | Ivan P.  | 2 pcs    | New    | [Accept] [X] |
| 2 | Maria D. | 1 pc     | Shipped| [View]       |
```

Status: **New** → **In Progress** → **Ready** → **Shipped**

---

## Launch

1. **Domain**: gravir.pro
2. **Hosting**: Shared or Vercel/Forge
3. **SSL**: Free (Let's Encrypt)

---

## After Launch

- [ ] Google Analytics — Traffic tracking
- [ ] Google Search Console — Indexing
- [ ] Facebook Pixel — Advertising

---

## Commands

```bash
# Install
composer install
npm install

# Run
php artisan serve
npm run dev

# Deploy
npm run build
php artisan migrate
php artisan db:seed
```

---

## What Comes After

After first 50 orders, add:

1. [ ] User accounts (login/register)
2. [ ] Customer reviews
3. [ ] Order tracking
4. [ ] Card payments (Stripe)
5. [ ] Product personalization (image upload)
6. [ ] Econt courier integration
