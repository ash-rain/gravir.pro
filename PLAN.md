# Gravir.pro — Technical Plan

## Overview

Build a luxurious e-commerce store for personalized wooden gifts. Simple but beautiful.

---

## What We Need

### Must Have
1. Products with luxury photos and prices
2. Cart
3. Checkout (name, phone, address)
4. Payment: Cash on Delivery + Bank Transfer + PayPal
5. Email confirmation
6. Filament admin panel

### What We Don't Need Now
- User accounts (optional via Breeze)
- Reviews
- Order tracking
- Multi-language (keep it for later)
- Complex design tool

---

## Pages

```
/                      → Home (luxury)
/products              → Products
/product/{slug}       → Product detail
/cart                 → Cart
/checkout             → Checkout
/thank-you            → Thank you
/admin                 → Filament panel
```

---

## Database (SQLite)

```php
// database/migrations/xxxx_xx_xx_create_tables.php

Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('phone', 20)->nullable();
    $table->text('address')->nullable();
    $table->string('city', 100)->nullable();
    $table->boolean('is_admin')->default(false);
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});

Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug')->unique();
    $table->text('description')->nullable();
    $table->string('image')->nullable();
    $table->timestamps();
});

Schema::create('products', function (Blueprint $table) {
    $table->id();
    $table->foreignId('category_id')->constrained();
    $table->string('name');
    $table->string('slug')->unique();
    $table->text('description');
    $table->decimal('price', 10, 2);
    $table->string('image');
    $table->json('images')->nullable();
    $table->boolean('is_active')->default(true);
    $table->integer('stock')->default(100);
    $table->timestamps();
});

Schema::create('orders', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->nullable()->constrained();
    $table->string('name');
    $table->string('phone', 20);
    $table->text('address');
    $table->string('city', 100);
    $table->string('payment_method'); // 'bank_transfer', 'cash_on_delivery', 'paypal'
    $table->decimal('subtotal', 10, 2);
    $table->decimal('shipping', 10, 2)->default(0);
    $table->decimal('total', 10, 2);
    $table->enum('status', ['new', 'confirmed', 'processing', 'ready', 'shipped', 'delivered', 'cancelled'])->default('new');
    $table->text('notes')->nullable();
    $table->string('tracking_number')->nullable();
    $table->timestamps();
});

Schema::create('order_items', function (Blueprint $table) {
    $table->id();
    $table->foreignId('order_id')->constrained()->cascadeOnDelete();
    $table->foreignId('product_id')->constrained();
    $table->integer('quantity')->default(1);
    $table->decimal('price', 10, 2);
    $table->text('customization')->nullable();
    $table->timestamps();
});
```

---

## Docker Setup (Simplified)

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    volumes:
      - .:/var/www/html
    environment:
      - APP_ENV=local
      - APP_URL=http://localhost:8000
      - DB_CONNECTION=sqlite
      - DB_DATABASE=/var/www/html/database/database.sqlite
    working_dir: /var/www/html
    command: >
      sh -c "
        echo 'Waiting for SQLite...' &&
        touch database/database.sqlite &&
        php artisan migrate --force &&
        php artisan db:seed --force &&
        php artisan serve --host=0.0.0.0 --port=8000
      "
```

```dockerfile
# Dockerfile
FROM php:8.5-cli

RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    oniguruma-dev \
    libxml2-dev \
    zip \
    unzip \
    libsqlite3-dev \
    sqlite3

RUN docker-php-ext-install pdo_sqlite mbstring exif pcntl

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

COPY . .

RUN composer install --no-dev --optimize-autoloader
RUN npm install && npm run build

EXPOSE 8000

CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

---

## Structure

```
app/
├── Filament/
│   └── Pages/
│       └── Dashboard.php
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
│   ├── Category.php
│   ├── Order.php
│   └── OrderItem.php
├── Providers/
│   └── AppServiceProvider.php
└── View/
    └── Components/
```

---

## Step by Step

### Week 1: Foundation
- [ ] Laravel project with Docker
- [ ] Database migrations (SQLite)
- [ ] Models (Product, Category, Order)
- [ ] Install Filament + Breeze
- [ ] Admin panel
- [ ] Home page (luxury design)

### Week 2: Products
- [ ] Product listing
- [ ] Product page
- [ ] Image upload
- [ ] Seed data

### Week 3: Cart + Checkout
- [ ] Cart (Livewire)
- [ ] Checkout form
- [ ] Payment: COD, Bank Transfer, PayPal
- [ ] Save to database

### Week 4: Admin + Launch
- [ ] Order management in Filament
- [ ] Status updates
- [ ] Email confirmation
- [ ] **LAUNCH**

---

## Demo Seeder — Full Demo Data

```php
// database/seeders/DatabaseSeeder.php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // Categories
        $categories = [
            ['name' => 'Wedding Gifts', 'slug' => 'wedding-gifts', 'description' => 'Personalized keepsakes for your special day', 'image' => 'categories/wedding.jpg'],
            ['name' => 'Name Plates', 'slug' => 'name-plates', 'description' => 'For doors or as a special gift', 'image' => 'categories/plates.jpg'],
            ['name' => 'Photo Engravings', 'slug' => 'photo-engravings', 'description' => 'Photos transformed into wooden art', 'image' => 'categories/photos.jpg'],
            ['name' => 'Home Decor', 'slug' => 'home-decor', 'description' => 'Functional art for your home', 'image' => 'categories/decor.jpg'],
            ['name' => 'Corporate', 'slug' => 'corporate', 'description' => 'Professional gifts for business', 'image' => 'categories/corporate.jpg'],
        ];

        foreach ($categories as $cat) {
            DB::table('categories')->insert([...$cat, 'created_at' => now(), 'updated_at' => now()]);
        }

        // Products - 15 products
        $products = [
            // Wedding
            ['category_id' => 1, 'name' => 'Wedding Plaque "Together"', 'slug' => 'wedding-plaque-together', 'description' => 'Romantic wedding plaque with couple names and date.', 'price' => 45.00, 'image' => 'products/wedding-plaque-1.jpg', 'stock' => 50],
            ['category_id' => 1, 'name' => 'Wooden Ring Box', 'slug' => 'wooden-ring-box', 'description' => 'Elegant wooden box for wedding rings.', 'price' => 35.00, 'image' => 'products/ring-box.jpg', 'stock' => 30],
            ['category_id' => 1, 'name' => 'Wedding Cutting Board', 'slug' => 'wedding-cutting-board', 'description' => 'Personalized board with names and monogram.', 'price' => 40.00, 'image' => 'products/wedding-board.jpg', 'stock' => 40],
            
            // Name Plates
            ['category_id' => 2, 'name' => 'Name Plate "Classic"', 'slug' => 'name-plate-classic', 'description' => 'Elegant name plate for your door.', 'price' => 18.00, 'image' => 'products/name-plate-1.jpg', 'stock' => 100],
            ['category_id' => 2, 'name' => 'Name Plate "Modern"', 'slug' => 'name-plate-modern', 'description' => 'Modern name plate with minimalist design.', 'price' => 22.00, 'image' => 'products/name-plate-modern.jpg', 'stock' => 80],
            ['category_id' => 2, 'name' => 'Kids Room Sign', 'slug' => 'kids-room-sign', 'description' => 'Fun sign for kids room.', 'price' => 15.00, 'image' => 'products/kids-sign.jpg', 'stock' => 60],
            
            // Photo Engravings
            ['category_id' => 3, 'name' => 'Photo Engraving "Memory"', 'slug' => 'photo-engraving-memory', 'description' => 'Transform your favorite photo into wooden art.', 'price' => 45.00, 'image' => 'products/photo-1.jpg', 'stock' => 40],
            ['category_id' => 3, 'name' => 'Pet Portrait', 'slug' => 'pet-portrait', 'description' => 'Turn your pet photo into lasting wood art.', 'price' => 35.00, 'image' => 'products/pet-portrait.jpg', 'stock' => 50],
            ['category_id' => 3, 'name' => 'Photo Coaster Set (4pc)', 'slug' => 'photo-coaster-set', 'description' => 'Set of 4 coasters with engraved photos.', 'price' => 30.00, 'image' => 'products/photo-coasters.jpg', 'stock' => 70],
            
            // Home Decor
            ['category_id' => 4, 'name' => 'Cutting Board "Chef"', 'slug' => 'cutting-board-chef', 'description' => 'Professional board with engraved recipe.', 'price' => 35.00, 'image' => 'products/cutting-board-1.jpg', 'stock' => 45],
            ['category_id' => 4, 'name' => 'Coaster Set (4pc)', 'slug' => 'coaster-set-4', 'description' => 'Elegant set of 4 wooden coasters.', 'price' => 22.00, 'image' => 'products/coaster-set.jpg', 'stock' => 100],
            ['category_id' => 4, 'name' => 'Phone Stand', 'slug' => 'phone-stand', 'description' => 'Stylish phone stand in wood.', 'price' => 18.00, 'image' => 'products/phone-stand.jpg', 'stock' => 80],
            
            // Corporate
            ['category_id' => 5, 'name' => 'Award "Champion"', 'slug' => 'award-first', 'description' => 'Personalized wooden award for employees.', 'price' => 40.00, 'image' => 'products/award-1.jpg', 'stock' => 30],
            ['category_id' => 5, 'name' => 'Business Gift Set', 'slug' => 'business-gift-set', 'description' => 'Set with company logo.', 'price' => 60.00, 'image' => 'products/business-set.jpg', 'stock' => 25],
            ['category_id' => 5, 'name' => 'Office Sign', 'slug' => 'office-sign', 'description' => 'Professional sign for office door.', 'price' => 25.00, 'image' => 'products/office-sign.jpg', 'stock' => 50],
        ];

        foreach ($products as $product) {
            DB::table('products')->insert([
                ...$product,
                'is_active' => true,
                'created_at' => now(),
                'updated_at' => now(),
            ]);
        }

        // Admin User
        DB::table('users')->insert([
            'name' => 'Admin',
            'email' => 'admin@gravir.pro',
            'phone' => '+359888123456',
            'is_admin' => true,
            'password' => bcrypt('admin123'),
            'created_at' => now(),
            'updated_at' => now(),
        ]);

        // Demo Orders
        $orders = [
            ['name' => 'Maria Petrova', 'phone' => '+359888111222', 'address' => '15 Gradinska St, ap. 4', 'city' => 'Sofia', 'payment_method' => 'cash_on_delivery', 'subtotal' => 45.00, 'shipping' => 5.00, 'total' => 50.00, 'status' => 'processing'],
            ['name' => 'Ivan Georgiev', 'phone' => '+359887333444', 'address' => '78 Vitosha Blvd', 'city' => 'Sofia', 'payment_method' => 'bank_transfer', 'subtotal' => 80.00, 'shipping' => 0.00, 'total' => 80.00, 'status' => 'confirmed'],
            ['name' => 'Nikoleta Todorova', 'phone' => '+359889555666', 'address' => 'Lyulin 8, bl.123', 'city' => 'Plovdiv', 'payment_method' => 'cash_on_delivery', 'subtotal' => 35.00, 'shipping' => 5.00, 'total' => 40.00, 'status' => 'shipped', 'tracking_number' => 'EONT123456789'],
        ];

        foreach ($orders as $order) {
            $orderId = DB::table('orders')->insertGetId([...$order, 'created_at' => now(), 'updated_at' => now()]);
            DB::table('order_items')->insert([
                'order_id' => $orderId,
                'product_id' => rand(1, 5),
                'quantity' => rand(1, 2),
                'price' => rand(15, 45),
                'created_at' => now(),
                'updated_at' => now(),
            ]);
        }
    }
}
```

---

## Payment Methods

### 1. Cash on Delivery (Наложен платеж)
- Popular in Bulgaria
- Customer pays to courier on delivery
- No upfront risk for customer

### 2. Bank Transfer (Банков превод)
- Account details shown after order
- Customer transfers manually
- We ship after confirmation

### 3. PayPal
- Link to PayPal.me or PayPal checkout
- Instant confirmation

```php
// In Checkout.php
public $payment_method = 'cash_on_delivery';

public function submit()
{
    $this->validate();
    
    $order = Order::create([
        'name' => $this->name,
        'phone' => $this->phone,
        'address' => $this->address,
        'city' => $this->city,
        'payment_method' => $this->payment_method, // 'bank_transfer', 'cash_on_delivery', 'paypal'
        'total' => $total,
        'status' => 'new',
    ]);
    
    return redirect('/thank-you');
}
```

---

## Filament Admin Panel

```php
// app/Filament/Resources/OrderResource.php
<?php

namespace App\Filament\Resources;

use Filament\Tables;
use Filament\Resources\Resource;
use App\Models\Order;

class OrderResource extends Resource
{
    protected static ?string $model = Order::class;
    protected static ?string $navigationIcon = 'heroicon-o-shopping-cart';
    protected static ?string $navigationLabel = 'Orders';

    public static function table(\Filament\Tables\Table $table): \Filament\Tables\Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('id')->label('#')->sortable(),
                Tables\Columns\TextColumn::make('name')->label('Customer')->searchable(),
                Tables\Columns\TextColumn::make('phone')->label('Phone'),
                Tables\Columns\TextColumn::make('city')->label('City'),
                Tables\Columns\TextColumn::make('total')->label('Total')->money('eur'),
                Tables\Columns\SelectColumn::make('status')
                    ->label('Status')
                    ->options([
                        'new' => 'New',
                        'confirmed' => 'Confirmed',
                        'processing' => 'Processing',
                        'ready' => 'Ready',
                        'shipped' => 'Shipped',
                        'delivered' => 'Delivered',
                        'cancelled' => 'Cancelled',
                    ]),
                Tables\Columns\TextColumn::make('created_at')->label('Date')->dateTime('d.m.Y H:i'),
            ])
            ->defaultSort('created_at', 'desc');
    }

    public static function getPages(): array
    {
        return [
            'index' => \App\Filament\Resources\OrderResource\Pages\ListOrders::route('/'),
        ];
    }
}
```

---

## Luxury Design

### Colors (like index.html)
- Dark background: #0a0a0a
- Gold: #b8954a
- Charcoal: #1f1f1f

### Tailwind Classes
```html
<div class="bg-charcoal-900 rounded-2xl border border-gold-500/10 hover:border-gold-500/30 transition-all">
    <img src="/storage/{{ $product->image }}" class="aspect-square object-cover" />
    <div class="p-6">
        <h3 class="font-serif text-xl text-white">{{ $product->name }}</h3>
        <span class="text-gold-500">from €{{ $product->price }}</span>
    </div>
</div>
```

---

## Bulgarian Translations (lang/bg)

```php
// lang/bg/messages.php
<?php

return [
    'home' => 'Начало',
    'products' => 'Продукти',
    'cart' => 'Кошница',
    'checkout' => 'Поръчка',
    'order' => 'Поръчка',
    'name' => 'Име',
    'phone' => 'Телефон',
    'address' => 'Адрес',
    'city' => 'Град',
    'total' => 'Общо',
    'submit' => 'Поръчай',
    'processing' => 'В обработка',
    'shipped' => 'Пратен',
    'delivered' => 'Доставен',
    'payment_bank_transfer' => 'Банков превод',
    'payment_cash_on_delivery' => 'Наложен платеж',
    'payment_paypal' => 'PayPal',
    'free_shipping' => 'Безплатна доставка',
    'from' => 'от',
];
```

```php
// lang/en/messages.php
<?php

return [
    'home' => 'Home',
    'products' => 'Products',
    'cart' => 'Cart',
    'checkout' => 'Checkout',
    'order' => 'Order',
    'name' => 'Name',
    'phone' => 'Phone',
    'address' => 'Address',
    'city' => 'City',
    'total' => 'Total',
    'submit' => 'Order',
    'processing' => 'Processing',
    'shipped' => 'Shipped',
    'delivered' => 'Delivered',
    'payment_bank_transfer' => 'Bank Transfer',
    'payment_cash_on_delivery' => 'Cash on Delivery',
    'payment_paypal' => 'PayPal',
    'free_shipping' => 'Free shipping',
    'from' => 'from',
];
```

---

## Commands

```bash
# Build and run
docker-compose up -d --build

# The container will auto:
# - Create SQLite database
# - Run migrations
# - Seed data
# - Start server on port 8000

# Access
# http://localhost:8000
# Admin: http://localhost:8000/admin
# Login: admin@gravir.pro / admin123
```

---

## What Comes After Launch

After first 50 orders, add:

1. [ ] User accounts (Breeze)
2. [ ] Customer reviews
3. [ ] Order tracking
4. [ ] Econt courier integration
5. [ ] SEO optimization
6. [ ] Multi-language (EN, BG)
