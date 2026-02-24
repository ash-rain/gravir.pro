# Gravir.pro — Технически План

## Само Най-Важното

Правим луксозен онлайн магазин за персонализирани дървени подаръци. Колкото по-просто, толкова по-добре.

---

## Какво Ни Трябва

### Задължително
1. Продукти с луксозни снимки и цени
2. Кошница
3. Поръчка (име, телефон, адрес)
4. Плащане (банков превод + Наложен платеж)
5. Потвърждение по имейл
6. Филament админ панел

### Кво Не Ни Трябва Сега
- Потребителски профили
- Отзиви
- Проследяване на поръчки
- Многоезична поддръжка
- Сложна кутия за дизайн

---

## Страници

```
/                      → Начало (луксозно)
/products              → Продукти
/product/{slug}       → Детайли за продукт
/cart                 → Кошница
/checkout             → Поръчка
/thank-you            → Благодаря
/admin                 → Филament панел
```

---

## База Данни

```php
// migrations/xxxx_xx_xx_create_tables.php

Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('phone', 20)->nullable();
    $table->text('address')->nullable();
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
    $table->json('images')->nullable(); // Gallery
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
    $table->string('payment_method'); // 'bank_transfer', 'cash_on_delivery'
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
    $table->text('customization')->nullable(); // JSON: {text, image}
    $table->timestamps();
});
```

---

## Технологии

```bash
# Инсталация
composer create-project laravel/laravel gravir --prefer-dist
cd gravir

# Основни пакети
composer require livewire/livewire
composer require filament/filament:"^5.0" --ignore-platform-reqs
composer require filament/forms:"^5.0" --ignore-platform-reqs
composer require filament/tables:"^5.0" --ignore-platform-reqs

# За снимки
composer require intervention/image

# За плащания
composer require stripe/stripe-php

# Frontend
npm install
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Пакети
- **Laravel 12** — Уеб рамка
- **Livewire 4** — Интерактивност без JavaScript
- **Filament 5** — Админ панел (луксозен!)
- **Tailwind CSS 4** — Стиловене
- **Stripe** — Плащания с карта
- **Intervention Image** — Обработка на снимки

---

## Docker Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:80"
    volumes:
      - .:/var/www/html
    depends_on:
      - db
      - redis
    environment:
      - APP_ENV=local
      - DB_HOST=db
      - DB_DATABASE=gravir
      - DB_USERNAME=gravir
      - DB_PASSWORD=gravir123
      - REDIS_HOST=redis

  db:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      - MYSQL_DATABASE=gravir
      - MYSQL_USER=gravir
      - MYSQL_PASSWORD=gravir123
      - MYSQL_ROOT_PASSWORD=root123
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - .:/var/www/html
    depends_on:
      - app

volumes:
  mysql_data:
```

```dockerfile
# Dockerfile
FROM php:8.2-fpm

RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    oniguruma-dev \
    libxml2-dev \
    zip \
    unzip

RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

COPY . .

RUN composer install --no-dev --optimize-autoloader
RUN npm install && npm run build

EXPOSE 80

CMD ["php-fpm"]
```

```nginx
# nginx.conf
server {
    listen 80;
    server_name localhost;
    root /var/www/html/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

---

## Структура

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

## Стъпка по Стъпка

### Седмица 1: Основа
- [ ] Laravel + Docker проект
- [ ] База данни + миграции
- [ ] Модели (Product, Category, Order)
- [ ] Filament админ панел
- [ ] Главна страница (луксозен дизайн)

### Седмица 2: Продукти
- [ ] Листинг на продукти
- [ ] Страница продукт
- [ ] Качване на снимки
- [ ] Seed данни

### Седмица 3: Кошница + Поръчка
- [ ] Кошница (Livewire)
- [ ] Форма за поръчка
- [ ] Плащане: банков превод + Наложен платеж
- [ ] Запазване в база

### Седмица 4: Админ + Пускане
- [ ] Управление на поръчки във Filament
- [ ] Промяна на статус
- [ ] Имейл потвърждение
- [ ] **ПУСКАНЕ**

---

## Demo Seeder — Пълни Демо Данни

```php
// database/seeders/DatabaseSeeder.php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // Categories
        $categories = [
            [
                'name' => 'Сватбени Подаръци',
                'slug' => 'wedding-gifts',
                'description' => 'Персонални спомени за най-важния ден',
                'image' => 'categories/wedding.jpg',
            ],
            [
                'name' => 'Именни Табели',
                'slug' => 'name-plates',
                'description' => 'За вратата или като специален подарък',
                'image' => 'categories/plates.jpg',
            ],
            [
                'name' => 'Фотогравировки',
                'slug' => 'photo-engravings',
                'description' => 'Снимки, превърнати в дървени произведения',
                'image' => 'categories/photos.jpg',
            ],
            [
                'name' => 'Домашни Декор',
                'slug' => 'home-decor',
                'description' => 'Функционално изкуство за вашия дом',
                'image' => 'categories/decor.jpg',
            ],
            [
                'name' => 'Корпоративни',
                'slug' => 'corporate',
                'description' => 'Професионални подаръци за бизнеса',
                'image' => 'categories/corporate.jpg',
            ],
        ];

        foreach ($categories as $category) {
            DB::table('categories')->insert([
                ...$category,
                'created_at' => now(),
                'updated_at' => now(),
            ]);
        }

        // Products
        $products = [
            // Сватбени Подаръци
            [
                'category_id' => 1,
                'name' => 'Сватбена Табела "Заедно"',
                'slug' => 'wedding-plaque-together',
                'description' => 'Романтична сватбена табела с имената на двойката и датата. Идеална за спомен от най-важния ден. Ръчна изработка от качествена дървесина.',
                'price' => 45.00,
                'image' => 'products/wedding-plaque-1.jpg',
                'images' => json_encode([
                    'products/wedding-plaque-1.jpg',
                    'products/wedding-plaque-2.jpg',
                    'products/wedding-plaque-3.jpg',
                ]),
                'stock' => 50,
            ],
            [
                'category_id' => 1,
                'name' => 'Дървена Кутия за Пръстени',
                'slug' => 'wooden-ring-box',
                'description' => 'Елегантна дървена кутия за сватбените пръстени. Гравирана с имената и датата на сватбата.',
                'price' => 35.00,
                'image' => 'products/ring-box.jpg',
                'images' => json_encode(['products/ring-box.jpg']),
                'stock' => 30,
            ],
            [
                'category_id' => 1,
                'name' => 'Сватбена Дъска за рязане',
                'slug' => 'wedding-cutting-board',
                'description' => 'Персонална дъска за рязане с гравирани имена и монограм. Функционален и красив подарък за новите младоженци.',
                'price' => 40.00,
                'image' => 'products/wedding-board.jpg',
                'images' => json_encode(['products/wedding-board.jpg']),
                'stock' => 40,
            ],

            // Именни Табели
            [
                'category_id' => 2,
                'name' => 'Именна Табела "Класика"',
                'slug' => 'name-plate-classic',
                'description' => 'Елегантна именна табела за вратата. Изберете шрифт и персонализирайте с името на семейството.',
                'price' => 18.00,
                'image' => 'products/name-plate-1.jpg',
                'images' => json_encode([
                    'products/name-plate-1.jpg',
                    'products/name-plate-2.jpg',
                ]),
                'stock' => 100,
            ],
            [
                'category_id' => 2,
                'name' => 'Именна Табела "Модерна"',
                'slug' => 'name-plate-modern',
                'description' => 'Модерна именна табела с минималистичен дизайн. Перфектна за модерни домове.',
                'price' => 22.00,
                'image' => 'products/name-plate-modern.jpg',
                'images' => json_encode(['products/name-plate-modern.jpg']),
                'stock' => 80,
            ],
            [
                'category_id' => 2,
                'name' => 'Табела за Детска Стая',
                'slug' => 'kids-room-sign',
                'description' => 'Забавна и цветна табела за детска стая. С име на детето и любими животни.',
                'price' => 15.00,
                'image' => 'products/kids-sign.jpg',
                'images' => json_encode(['products/kids-sign.jpg']),
                'stock' => 60,
            ],

            // Фотогравировки
            [
                'category_id' => 3,
                'name' => 'Фотогравировка "Спомен"',
                'slug' => 'photo-engraving-memory',
                'description' => 'Преобразувайте любима снимка в дървена гравировка. Перфектен подарък за най-близките хора.',
                'price' => 45.00,
                'image' => 'products/photo-1.jpg',
                'images' => json_encode([
                    'products/photo-1.jpg',
                    'products/photo-2.jpg',
                ]),
                'stock' => 40,
            ],
            [
                'category_id' => 3,
                'name' => 'Фотогравировка Домашен Любимец',
                'slug' => 'pet-portrait',
                'description' => 'Превърнете снимката на вашия любимец в трайно дървено произведение. Спомен, който трае завинаги.',
                'price' => 35.00,
                'image' => 'products/pet-portrait.jpg',
                'images' => json_encode(['products/pet-portrait.jpg']),
                'stock' => 50,
            ],
            [
                'category_id' => 3,
                'name' => 'Фотокомплект Подложки (4бр)',
                'slug' => 'photo-coaster-set',
                'description' => 'Комплект от 4 подложки с гравирани снимки. Може да са различни или еднакви.',
                'price' => 30.00,
                'image' => 'products/photo-coasters.jpg',
                'images' => json_encode(['products/photo-coasters.jpg']),
                'stock' => 70,
            ],

            // Домашни Декор
            [
                'category_id' => 4,
                'name' => 'Дъска за Рязане "Шеф"',
                'slug' => 'cutting-board-chef',
                'description' => 'Професионална дъска за рязане с гравирана рецепта или послание. Идеална за кулинари.',
                'price' => 35.00,
                'image' => 'products/cutting-board-1.jpg',
                'images' => json_encode(['products/cutting-board-1.jpg']),
                'stock' => 45,
            ],
            [
                'category_id' => 4,
                'name' => 'Комплект Подложки (4бр)',
                'slug' => 'coaster-set-4',
                'description' => 'Елегантен комплект от 4 дървени подложки с различни мотиви.',
                'price' => 22.00,
                'image' => 'products/coaster-set.jpg',
                'images' => json_encode(['products/coaster-set.jpg']),
                'stock' => 100,
            ],
            [
                'category_id' => 4,
                'name' => 'Дървена Кутия за Бижута',
                'slug' => 'jewelry-box',
                'description' => 'Ръчно изработена дървена кутия за бижута с гравиран мотив.',
                'price' => 28.00,
                'image' => 'products/jewelry-box.jpg',
                'images' => json_encode(['products/jewelry-box.jpg']),
                'stock' => 35,
            ],
            [
                'category_id' => 4,
                'name' => 'Поставка за Телефон',
                'slug' => 'phone-stand',
                'description' => 'Стильна поставка за телефон от дърво. Практичен и красив декор за бюрото.',
                'price' => 18.00,
                'image' => 'products/phone-stand.jpg',
                'images' => json_encode(['products/phone-stand.jpg']),
                'stock' => 80,
            ],
            [
                'category_id' => 4,
                'name' => 'Дървено Пано "Природа"',
                'slug' => 'wood-art-nature',
                'description' => 'Художествено пано от финер с гравиран пейзаж или мото.',
                'price' => 55.00,
                'image' => 'products/wood-art.jpg',
                'images' => json_encode(['products/wood-art.jpg']),
                'stock' => 20,
            ],

            // Корпоративни
            [
                'category_id' => 5,
                'name' => 'Награда "Първенец"',
                'slug' => 'award-first',
                'description' => 'Персонална дървена награда за служители или партньори.',
                'price' => 40.00,
                'image' => 'products/award-1.jpg',
                'images' => json_encode(['products/award-1.jpg']),
                'stock' => 30,
            ],
            [
                'category_id' => 5,
                'name' => 'Бизнес Подарък Комплект',
                'slug' => 'business-gift-set',
                'description' => 'Комплект от дъска за рязане и подложки с лого на компанията.',
                'price' => 60.00,
                'image' => 'products/business-set.jpg',
                'images' => json_encode(['products/business-set.jpg']),
                'stock' => 25,
            ],
            [
                'category_id' => 5,
                'name' => 'Табела за Офис',
                'slug' => 'office-sign',
                'description' => 'Професионална табела за офис врата с име и длъжност.',
                'price' => 25.00,
                'image' => 'products/office-sign.jpg',
                'images' => json_encode(['products/office-sign.jpg']),
                'stock' => 50,
            ],
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
        $demoOrders = [
            [
                'name' => 'Мария Петрова',
                'phone' => '+359888111222',
                'address' => 'ул. "Градинска" 15, ап. 4',
                'city' => 'София',
                'payment_method' => 'cash_on_delivery',
                'subtotal' => 45.00,
                'shipping' => 5.00,
                'total' => 50.00,
                'status' => 'processing',
                'notes' => 'Моля, обадете се преди доставка',
            ],
            [
                'name' => 'Иван Георгиев',
                'phone' => '+359887333444',
                'address' => 'бул. "Витоша" 78',
                'city' => 'София',
                'payment_method' => 'bank_transfer',
                'subtotal' => 80.00,
                'shipping' => 0.00,
                'total' => 80.00,
                'status' => 'confirmed',
                'notes' => '',
            ],
            [
                'name' => 'Николета Тодорова',
                'phone' => '+359889555666',
                'address' => 'ж.к. "Люлин" 8, бл. 123, вх. Б',
                'city' => 'Пловдив',
                'payment_method' => 'cash_on_delivery',
                'subtotal' => 35.00,
                'shipping' => 5.00,
                'total' => 40.00,
                'status' => 'shipped',
                'tracking_number' => 'EONT123456789',
                'notes' => 'Доставка до офис на Спиди',
            ],
        ];

        foreach ($demoOrders as $order) {
            $orderId = DB::table('orders')->insertGetId([
                ...$order,
                'created_at' => now(),
                'updated_at' => now(),
            ]);

            // Add order items
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

## Filament Админ Панел

```php
// app/Filament/Resources/OrderResource.php
<?php

namespace App\Filament\Resources;

use Filament\Forms;
use Filament\Tables;
use Filament\Resources\Resource;
use Filament\Tables\Table;
use App\Filament\Resources\OrderResource\Pages;
use App\Models\Order;

class OrderResource extends Resource
{
    protected static ?string $model = Order::class;

    protected static ?string $navigationIcon = 'heroicon-o-shopping-cart';

    protected static ?string $navigationLabel = 'Поръчки';

    protected static ?string $modelLabel = 'Поръчка';

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('id')
                    ->label('#')
                    ->sortable(),
                Tables\Columns\TextColumn::make('name')
                    ->label('Клиент')
                    ->searchable(),
                Tables\Columns\TextColumn::make('phone')
                    ->label('Телефон'),
                Tables\Columns\TextColumn::make('city')
                    ->label('Град'),
                Tables\Columns\TextColumn::make('total')
                    ->label('Сума')
                    ->money('eur'),
                Tables\Columns\SelectColumn::make('status')
                    ->label('Статус')
                    ->options([
                        'new' => 'Нова',
                        'confirmed' => 'Потвърдена',
                        'processing' => 'В обработка',
                        'ready' => 'Готова',
                        'shipped' => 'Пратена',
                        'delivered' => 'Доставена',
                        'cancelled' => 'Отказана',
                    ])
                    ->updateStateUsing(fn ($record, $state) => $record->update(['status' => $state])),
                Tables\Columns\TextColumn::make('created_at')
                    ->label('Дата')
                    ->dateTime('d.m.Y H:i')
                    ->sortable(),
            ])
            ->defaultSort('created_at', 'desc')
            ->actions([
                Tables\Actions\ViewAction::make(),
                Tables\Actions\EditAction::make(),
            ])
            ->bulkActions([
                Tables\Actions\BulkActionGroup::make([
                    Tables\Actions\DeleteBulkAction::make(),
                ]),
            ]);
    }

    public static function getPages(): array
    {
        return [
            'index' => Pages\ListOrders::route('/'),
            'view' => Pages\ViewOrder::route('/{record}'),
        ];
    }
}
```

---

## Checkout Livewire с Плащане

```php
// app/Livewire/Checkout.php
<?php

namespace App\Livewire;

use Livewire\Component;
use App\Models\Product;
use App\Models\Order;
use App\Models\OrderItem;

class Checkout extends Component
{
    public $name = '';
    public $phone = '';
    public $address = '';
    public $city = '';
    public $payment_method = 'cash_on_delivery';
    public $notes = '';
    
    protected $rules = [
        'name' => 'required|min:2',
        'phone' => 'required|min:10',
        'address' => 'required',
        'city' => 'required',
        'payment_method' => 'required|in:bank_transfer,cash_on_delivery',
    ];
    
    public function submit()
    {
        $this->validate();
        
        $cart = session()->get('cart', []);
        
        if (empty($cart)) {
            return redirect('/products');
        }
        
        $subtotal = collect($cart)->sum('total');
        $shipping = $subtotal >= 50 ? 0 : 5; // Безплатна над €50
        $total = $subtotal + $shipping;
        
        $order = Order::create([
            'name' => $this->name,
            'phone' => $this->phone,
            'address' => $this->address,
            'city' => $this->city,
            'payment_method' => $this->payment_method,
            'subtotal' => $subtotal,
            'shipping' => $shipping,
            'total' => $total,
            'notes' => $this->notes,
            'status' => 'new',
        ]);
        
        foreach ($cart as $item) {
            OrderItem::create([
                'order_id' => $order->id,
                'product_id' => $item['product_id'],
                'quantity' => $item['quantity'],
                'price' => $item['price'],
                'customization' => $item['customization'] ?? null,
            ]);
        }
        
        session()->forget('cart');
        
        // Изпращане на имейл...
        
        return redirect('/thank-you?order=' . $order->id);
    }
    
    public function render()
    {
        $cart = session()->get('cart', []);
        $subtotal = collect($cart)->sum('total');
        $shipping = $subtotal >= 50 ? 0 : 5;
        $total = $subtotal + $shipping;
        
        return view('livewire.checkout', [
            'cart' => $cart,
            'subtotal' => $subtotal,
            'shipping' => $shipping,
            'total' => $total,
        ]);
    }
}
```

---

## Плащане при Получаване (Cash on Delivery)

### Защо е важно
- Българите предпочитат да платят при получаване
- Намалява риска за клиента
- Увеличава конверсията

### Как работи
```
1. Клиент избира "Наложен платеж"
2. Поръчката се потвърждава автоматично
3. Изпращаме с Econt/Speedy с опция за преглед
4. Клиент плаща на куриера при получаване
5. Ние получаваме парите от куриера
```

### Код
```php
// В модела Order
public function isCashOnDelivery(): bool
{
    return $this->payment_method === 'cash_on_delivery';
}

public function getPaymentStatusAttribute(): string
{
    if ($this->isCashOnDelivery()) {
        return match($this->status) {
            'delivered' => 'paid',
            default => 'pending',
        };
    }
    
    return match($this->status) {
        'confirmed', 'processing', 'ready', 'shipped', 'delivered' => 'paid',
        default => 'pending',
    };
}
```

---

## Дизайн (Луксозен)

### Принципи (като index.html)
- Тъмен фон (#0a0a0a)
- Златни акценти (#b8954a)
- Много бяло пространство
- Една главна снимка на продукт
- Ясни бутони
- Бързо зареждане
- За мобилни

### Цветове
```css
--color-charcoal-900: #0a0a0a;
--color-charcoal-800: #141414;
--color-charcoal-700: #1f1f1f;
--color-gold-500: #b8954a;
--color-gold-400: #c9a96f;
--color-gold-300: #d9c49e;
```

### Tailwind класове
```html
<!-- Карта продукт -->
<div class="bg-charcoal-800 rounded-2xl border border-gold-500/10 hover:border-gold-500/30 transition-all duration-500">
    <div class="aspect-square bg-gradient-to-br from-charcoal-700 to-charcoal-800">
        <img src="/storage/{{ $product->image }}" alt="{{ $product->name }}" />
    </div>
    <div class="p-6">
        <h3 class="font-serif text-xl text-white">{{ $product->name }}</h3>
        <p class="text-gray-400 text-sm">{{ $product->description }}</p>
        <span class="text-gold-500 font-medium">от €{{ $product->price }}</span>
    </div>
</div>
```

---

## Снимки

Снимките са всичко. Всеки продукт има 1-3 луксозни снимки.

- Фон: Тъмен или бял
- Светлина: Професионална
- Ъгъл: Равно отгоре или 45°
- Резолюция: 1200x1200
- Формат: WebP

---

## Команди

```bash
# Инсталиране
docker-compose up -d --build
docker-compose exec app composer install
docker-compose exec app php artisan migrate --seed
docker-compose exec app npm install

# Стартиране
docker-compose up -d

# Миграции
docker-compose exec app php artisan migrate

# Seed
docker-compose exec app php artisan db:seed

# Логове
docker-compose logs -f app
```

---

## Какво След Пускане

След първите 50 поръчки добавяме:

1. [ ] Потребителски профили (вход/регистрация)
2. [ ] Отзиви от клиенти
3. [ ] Проследяване на поръчка
4. [ ] Плащане с карта (Stripe)
5. [ ] Econt куриер интеграция
6. [ ] SEO оптимизация
