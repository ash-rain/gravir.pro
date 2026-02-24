# Gravir.pro — Technical Plan

## Overview

Build a luxurious e-commerce store for personalized wooden gifts. Multi-language from the start.

---

## What We Need

### Must Have
1. Products with luxury photos and prices
2. Cart
3. Checkout (name, phone, address)
4. Payment: Cash on Delivery + Bank Transfer + PayPal
5. Email confirmation
6. Filament admin panel
7. **Multi-language (EN, BG default + EU languages)**

### What We Don't Need Now
- User accounts (optional via Breeze)
- Reviews
- Order tracking

---

## Pages

```
/                      → Home (luxury)
/{locale}/             → Home localized (en, bg, de, fr, es, it, ro, gr)
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
    $table->string('locale')->default('bg'); // Default language
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});

Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('name'); // JSON: {'en': 'Wedding', 'bg': 'Сватбени'}
    $table->string('slug')->unique();
    $table->text('description'); // JSON translations
    $table->string('image')->nullable();
    $table->timestamps();
});

Schema::create('products', function (Blueprint $table) {
    $table->id();
    $table->foreignId('category_id')->constrained();
    $table->string('name'); // JSON: {'en': 'Name', 'bg': 'Име'}
    $table->string('slug')->unique();
    $table->text('description'); // JSON translations
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
    $table->string('locale')->default('bg'); // Order language
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

## Multi-Language Strategy

### Supported Languages
| Code | Language | Default |
|------|----------|---------|
| bg | Bulgarian | ✅ Yes |
| en | English | |
| de | German | |
| fr | French | |
| es | Spanish | |
| it | Italian | |
| ro | Romanian | |
| gr | Greek | |

### Implementation
- **URL Structure**: `gravir.pro/bg/products`, `gravir.pro/en/products`
- **Default**: Bulgarian (`bg`)
- **Storage**: JSON fields in database for translations
- **Fallback**: If translation missing → show Bulgarian

### Language Switcher
```php
// In layout
<select onchange="window.location.href='/' + this.value">
    <option value="bg" {{ request()->locale() == 'bg' ? 'selected' : '' }}>Български</option>
    <option value="en" {{ request()->locale() == 'en' ? 'selected' : '' }}>English</option>
    <option value="de" {{ request()->locale() == 'de' ? 'selected' : '' }}>Deutsch</option>
    <option value="fr" {{ request()->locale() == 'fr' ? 'selected' : '' }}>Français</option>
    <option value="es" {{ request()->locale() == 'es' ? 'selected' : '' }}>Español</option>
    <option value="it" {{ request()->locale() == 'it' ? 'selected' : '' }}>Italiano</option>
    <option value="ro" {{ request()->locale() == 'ro' ? 'selected' : '' }}>Română</option>
    <option value="gr" {{ request()->locale() == 'gr' ? 'selected' : '' }}>Ελληνικά</option>
</select>
```

---

## Translations Helper

```php
// app/Helpers/Translatable.php
<?php

function trans_field($field, ?string $locale = null): string
{
    $locale = $locale ?? app()->getLocale();
    $data = json_decode($field, true);
    
    if (!$data) return $field;
    
    return $data[$locale] ?? $data['bg'] ?? $field;
}
```

```php
// Usage in Blade
<h1>{{ trans_field($product->name) }}</h1>
<p>{{ trans_field($product->description) }}</p>
```

---

## Language Files (lang/)

```php
// lang/bg/messages.php
<?php

return [
    // Navigation
    'home' => 'Начало',
    'products' => 'Продукти',
    'cart' => 'Кошница',
    'checkout' => 'Поръчка',
    'about' => 'За нас',
    'contact' => 'Контакти',
    
    // Checkout
    'name' => 'Име',
    'phone' => 'Телефон',
    'address' => 'Адрес',
    'city' => 'Град',
    'notes' => 'Бележки',
    'submit_order' => 'Поръчай',
    
    // Payment
    'payment_method' => 'Начин на плащане',
    'bank_transfer' => 'Банков превод',
    'cash_on_delivery' => 'Наложен платеж',
    'paypal' => 'PayPal',
    
    // Status
    'status_new' => 'Нова',
    'status_confirmed' => 'Потвърдена',
    'status_processing' => 'В обработка',
    'status_ready' => 'Готова',
    'status_shipped' => 'Пратена',
    'status_delivered' => 'Доставена',
    'status_cancelled' => 'Отказана',
    
    // Cart
    'cart_empty' => 'Кошницата е празна',
    'add_to_cart' => 'Добави в кошница',
    'remove' => 'Премахни',
    'quantity' => 'Количество',
    'subtotal' => 'Междинна сума',
    'shipping' => 'Доставка',
    'total' => 'Общо',
    'free_shipping' => 'Безплатна доставка над €50',
    
    // Products
    'from' => 'от',
    'in_stock' => 'В наличност',
    'out_of_stock' => 'Изчерпано',
    
    // General
    'thank_you' => 'Благодарим!',
    'order_received' => 'Поръчката е приета.',
    'we_will_contact' => 'Ще се свържем с вас скоро.',
    
    // Footer
    'all_rights_reserved' => 'Всички права запазени.',
    'made_in_bulgaria' => 'Ръчна изработка в България.',
];
```

```php
// lang/en/messages.php
<?php

return [
    // Navigation
    'home' => 'Home',
    'products' => 'Products',
    'cart' => 'Cart',
    'checkout' => 'Checkout',
    'about' => 'About',
    'contact' => 'Contact',
    
    // Checkout
    'name' => 'Name',
    'phone' => 'Phone',
    'address' => 'Address',
    'city' => 'City',
    'notes' => 'Notes',
    'submit_order' => 'Order Now',
    
    // Payment
    'payment_method' => 'Payment Method',
    'bank_transfer' => 'Bank Transfer',
    'cash_on_delivery' => 'Cash on Delivery',
    'paypal' => 'PayPal',
    
    // Status
    'status_new' => 'New',
    'status_confirmed' => 'Confirmed',
    'status_processing' => 'Processing',
    'status_ready' => 'Ready',
    'status_shipped' => 'Shipped',
    'status_delivered' => 'Delivered',
    'status_cancelled' => 'Cancelled',
    
    // Cart
    'cart_empty' => 'Your cart is empty',
    'add_to_cart' => 'Add to Cart',
    'remove' => 'Remove',
    'quantity' => 'Quantity',
    'subtotal' => 'Subtotal',
    'shipping' => 'Shipping',
    'total' => 'Total',
    'free_shipping' => 'Free shipping over €50',
    
    // Products
    'from' => 'from',
    'in_stock' => 'In Stock',
    'out_of_stock' => 'Out of Stock',
    
    // General
    'thank_you' => 'Thank You!',
    'order_received' => 'Your order has been received.',
    'we_will_contact' => 'We will contact you soon.',
    
    // Footer
    'all_rights_reserved' => 'All rights reserved.',
    'made_in_bulgaria' => 'Handmade in Bulgaria.',
];
```

```php
// lang/de/messages.php
<?php

return [
    'home' => 'Startseite',
    'products' => 'Produkte',
    'cart' => 'Warenkorb',
    'checkout' => 'Kasse',
    'payment_method' => 'Zahlungsmethode',
    'bank_transfer' => 'Überweisung',
    'cash_on_delivery' => 'Nachnahme',
    'paypal' => 'PayPal',
    'total' => 'Gesamt',
    'from' => 'ab',
];
```

```php
// lang/fr/messages.php
<?php

return [
    'home' => 'Accueil',
    'products' => 'Produits',
    'cart' => 'Panier',
    'checkout' => 'Commander',
    'payment_method' => 'Mode de paiement',
    'bank_transfer' => 'Virement bancaire',
    'cash_on_delivery' => 'Contre remboursement',
    'paypal' => 'PayPal',
    'total' => 'Total',
    'from' => 'à partir de',
];
```

```php
// lang/es/messages.php
<?php

return [
    'home' => 'Inicio',
    'products' => 'Productos',
    'cart' => 'Carrito',
    'checkout' => 'Pagar',
    'payment_method' => 'Método de pago',
    'bank_transfer' => 'Transferencia bancaria',
    'cash_on_delivery' => 'Contra reembolso',
    'paypal' => 'PayPal',
    'total' => 'Total',
    'from' => 'desde',
];
```

```php
// lang/it/messages.php
<?php

return [
    'home' => 'Home',
    'products' => 'Prodotti',
    'cart' => 'Carrello',
    'checkout' => 'Checkout',
    'payment_method' => 'Metodo di pagamento',
    'bank_transfer' => 'Bonifico bancario',
    'cash_on_delivery' => 'Contrassegno',
    'paypal' => 'PayPal',
    'total' => 'Totale',
    'from' => 'da',
];
```

```php
// lang/ro/messages.php
<?php

return [
    'home' => 'Acasă',
    'products' => 'Produse',
    'cart' => 'Coș',
    'checkout' => 'Comandă',
    'payment_method' => 'Metoda de plată',
    'bank_transfer' => 'Transfer bancar',
    'cash_on_delivery' => 'Ramburs',
    'paypal' => 'PayPal',
    'total' => 'Total',
    'from' => 'de la',
];
```

```php
// lang/gr/messages.php
<?php

return [
    'home' => 'Αρχική',
    'products' => 'Προϊόντα',
    'cart' => 'Καλάθι',
    'checkout' => 'Ολοκλήρωση',
    'payment_method' => 'Μέθοδος πληρωμής',
    'bank_transfer' => 'Τραπεζική μεταφορά',
    'cash_on_delivery' => 'Αντικαταβολή',
    'paypal' => 'PayPal',
    'total' => 'Σύνολο',
    'from' => 'από',
];
```

---

## Demo Seeder — Full Multi-Language Data

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
        // Categories - Multi-language
        $categories = [
            [
                'name' => json_encode([
                    'bg' => 'Сватбени Подаръци',
                    'en' => 'Wedding Gifts',
                    'de' => 'Hochzeitsgeschenke',
                    'fr' => 'Cadeaux de mariage',
                    'es' => 'Regalos de boda',
                    'it' => 'Regali di nozze',
                    'ro' => 'Cadouiri de nuntă',
                    'gr' => 'Δώρα γάμου',
                ]),
                'slug' => 'wedding-gifts',
                'description' => json_encode([
                    'bg' => 'Персонални спомени за най-важния ден',
                    'en' => 'Personalized keepsakes for your special day',
                    'de' => 'Persönliche Andenken für Ihren großen Tag',
                    'fr' => 'Souvenirs personnalisés pour votre grand jour',
                    'es' => 'Recuerdos personalizados para tu gran día',
                    'it' => 'Ricordi personalizzati per il tuo grande giorno',
                    'ro' => 'Amintiri personalizate pentru ziua ta specială',
                    'gr' => 'Εξατομικευμένα αναμνηστικά για τη μεγάλη σου μέρα',
                ]),
                'image' => 'categories/wedding.jpg',
            ],
            [
                'name' => json_encode([
                    'bg' => 'Именни Табели',
                    'en' => 'Name Plates',
                    'de' => 'Namensschilder',
                    'fr' => 'Plaques nominatives',
                    'es' => 'Placas con nombre',
                    'it' => 'Targhe con nome',
                    'ro' => 'Plăcuțe cu nume',
                    'gr' => 'Πινακίδες ονόματος',
                ]),
                'slug' => 'name-plates',
                'description' => json_encode([
                    'bg' => 'За вратата или като специален подарък',
                    'en' => 'For doors or as a special gift',
                    'de' => 'Für Türen oder als besonderes Geschenk',
                    'fr' => 'Pour les portes ou comme cadeau spécial',
                    'es' => 'Para puertas o como regalo especial',
                    'it' => 'Per porte o come regalo speciale',
                    'ro' => 'Pentru uși sau ca cadou special',
                    'gr' => 'Για πόρτες ή ως ειδικό δώρο',
                ]),
                'image' => 'categories/plates.jpg',
            ],
            [
                'name' => json_encode([
                    'bg' => 'Фотогравировки',
                    'en' => 'Photo Engravings',
                    'de' => 'Fotogravuren',
                    'fr' => 'Gravures photo',
                    'es' => 'Grabados foto',
                    'it' => 'Incisioni foto',
                    'ro' => 'Gravuri foto',
                    'gr' => 'Φωτογραφίες χάραξης',
                ]),
                'slug' => 'photo-engravings',
                'description' => json_encode([
                    'bg' => 'Снимки, превърнати в дървени произведения',
                    'en' => 'Photos transformed into wooden art',
                    'de' => 'Fotos in Holz Kunst verwandelt',
                    'fr' => 'Photos transformées en art bois',
                    'es' => 'Fotos transformadas en arte de madera',
                    'it' => 'Foto trasformate in arte legno',
                    'ro' => 'Fotografii transformate în artă din lemn',
                    'gr' => 'Φωτογραφίες μετατρέπονται σε ξύλινη τέχνη',
                ]),
                'image' => 'categories/photos.jpg',
            ],
            [
                'name' => json_encode([
                    'bg' => 'Домашен Декор',
                    'en' => 'Home Decor',
                    'de' => 'Wohndekoration',
                    'fr' => 'Décoration maison',
                    'es' => 'Decoración del hogar',
                    'it' => 'Arredamento casa',
                    'ro' => 'Decor locuință',
                    'gr' => 'Διακόσμηση σπιτιού',
                ]),
                'slug' => 'home-decor',
                'description' => json_encode([
                    'bg' => 'Функционално изкуство за вашия дом',
                    'en' => 'Functional art for your home',
                    'de' => 'Funktionelle Kunst für Ihr Zuhause',
                    'fr' => 'Art fonctionnel pour votre maison',
                    'es' => 'Arte funcional para tu hogar',
                    'it' => 'Arte funzionale per la tua casa',
                    'ro' => 'Artă funcțională pentru casa ta',
                    'gr' => 'Λειτουργική τέχνη για το σπίτι σου',
                ]),
                'image' => 'categories/decor.jpg',
            ],
            [
                'name' => json_encode([
                    'bg' => 'Корпоративни',
                    'en' => 'Corporate',
                    'de' => 'Unternehmen',
                    'fr' => 'Entreprise',
                    'es' => 'Corporativo',
                    'it' => 'Aziendale',
                    'ro' => 'Corporativ',
                    'gr' => 'Εταιρικά',
                ]),
                'slug' => 'corporate',
                'description' => json_encode([
                    'bg' => 'Професионални подаръци за бизнеса',
                    'en' => 'Professional gifts for business',
                    'de' => 'Professionelle Geschenke für Unternehmen',
                    'fr' => 'Cadeaux professionnels pour les entreprises',
                    'es' => 'Regalos profesionales para negocios',
                    'it' => 'Regali professionali per aziende',
                    'ro' => 'Cadouiri profesionale pentru afaceri',
                    'gr' => 'Επαγγελματικά δώρα για επιχειρήσεις',
                ]),
                'image' => 'categories/corporate.jpg',
            ],
        ];

        foreach ($categories as $cat) {
            DB::table('categories')->insert([...$cat, 'created_at' => now(), 'updated_at' => now()]);
        }

        // Products - Multi-language
        $products = [
            // Wedding
            [
                'category_id' => 1,
                'name' => json_encode(['bg' => 'Сватбена Табела "Заедно"', 'en' => 'Wedding Plaque "Together"', 'de' => 'Hochzeitsschild "Zusammen"']),
                'slug' => 'wedding-plaque-together',
                'description' => json_encode(['bg' => 'Романтична сватбена табела с имената на двойката и датата.', 'en' => 'Romantic wedding plaque with couple names and date.', 'de' => 'Romantisches Hochzeitsschild mit Namen und Datum.']),
                'price' => 45.00,
                'image' => 'products/wedding-plaque-1.jpg',
                'stock' => 50,
            ],
            [
                'category_id' => 1,
                'name' => json_encode(['bg' => 'Дървена Кутия за Пръстени', 'en' => 'Wooden Ring Box', 'de' => 'Holzringdose']),
                'slug' => 'wooden-ring-box',
                'description' => json_encode(['bg' => 'Елегантна дървена кутия за сватбените пръстени.', 'en' => 'Elegant wooden box for wedding rings.', 'de' => 'Elegante Holzkiste für Trauringe.']),
                'price' => 35.00,
                'image' => 'products/ring-box.jpg',
                'stock' => 30,
            ],
            [
                'category_id' => 1,
                'name' => json_encode(['bg' => 'Сватбена Дъска за рязане', 'en' => 'Wedding Cutting Board', 'de' => 'Hochzeits Schneidebrett']),
                'slug' => 'wedding-cutting-board',
                'description' => json_encode(['bg' => 'Персонална дъска с гравирани имена и монограм.', 'en' => 'Personalized board with engraved names and monogram.', 'de' => 'Personalisierte Tafel mit gravierten Namen und Monogramm.']),
                'price' => 40.00,
                'image' => 'products/wedding-board.jpg',
                'stock' => 40,
            ],
            
            // Name Plates
            [
                'category_id' => 2,
                'name' => json_encode(['bg' => 'Именна Табела "Класика"', 'en' => 'Name Plate "Classic"', 'de' => 'Namensschild "Klassik"']),
                'slug' => 'name-plate-classic',
                'description' => json_encode(['bg' => 'Елегантна именна табела за вратата.', 'en' => 'Elegant name plate for your door.', 'de' => 'Elegantes Namensschild für Ihre Tür.']),
                'price' => 18.00,
                'image' => 'products/name-plate-1.jpg',
                'stock' => 100,
            ],
            [
                'category_id' => 2,
                'name' => json_encode(['bg' => 'Именна Табела "Модерна"', 'en' => 'Name Plate "Modern"', 'de' => 'Namensschild "Modern"']),
                'slug' => 'name-plate-modern',
                'description' => json_encode(['bg' => 'Модерна именна табела с минималистичен дизайн.', 'en' => 'Modern name plate with minimalist design.', 'de' => 'Modernes Namensschild mit minimalistischem Design.']),
                'price' => 22.00,
                'image' => 'products/name-plate-modern.jpg',
                'stock' => 80,
            ],
            
            // Photo Engravings
            [
                'category_id' => 3,
                'name' => json_encode(['bg' => 'Фотогравировка "Спомен"', 'en' => 'Photo Engraving "Memory"', 'de' => 'Fotogravur "Erinnerung"']),
                'slug' => 'photo-engraving-memory',
                'description' => json_encode(['bg' => 'Преобразувайте любима снимка в дървена гравировка.', 'en' => 'Transform your favorite photo into wooden art.', 'de' => 'Verwandeln Sie Ihr Lieblingsfoto in Holzgravur.']),
                'price' => 45.00,
                'image' => 'products/photo-1.jpg',
                'stock' => 40,
            ],
            [
                'category_id' => 3,
                'name' => json_encode(['bg' => 'Фотогравировка Домашен Любимец', 'en' => 'Pet Portrait', 'de' => 'Tierporträt']),
                'slug' => 'pet-portrait',
                'description' => json_encode(['bg' => 'Превърнете снимката на вашия любимец в трайно дървено изкуство.', 'en' => 'Turn your pet photo into lasting wood art.', 'de' => 'Verwandeln Sie Ihr Haustierfoto in dauerhafte Holzkunst.']),
                'price' => 35.00,
                'image' => 'products/pet-portrait.jpg',
                'stock' => 50,
            ],
            
            // Home Decor
            [
                'category_id' => 4,
                'name' => json_encode(['bg' => 'Дъска за рязане "Шеф"', 'en' => 'Cutting Board "Chef"', 'de' => 'Schneidebrett "Koch"']),
                'slug' => 'cutting-board-chef',
                'description' => json_encode(['bg' => 'Професионална дъска с гравирана рецепта.', 'en' => 'Professional board with engraved recipe.', 'de' => 'Professionelles Brett mit graviertem Rezept.']),
                'price' => 35.00,
                'image' => 'products/cutting-board-1.jpg',
                'stock' => 45,
            ],
            [
                'category_id' => 4,
                'name' => json_encode(['bg' => 'Комплект Подложки (4бр)', 'en' => 'Coaster Set (4pc)', 'de' => 'Untersetzer Set (4St)']),
                'slug' => 'coaster-set-4',
                'description' => json_encode(['bg' => 'Елегантен комплект от 4 дървени подложки.', 'en' => 'Elegant set of 4 wooden coasters.', 'de' => 'Elegantes Set aus 4 Holzuntersetzer.']),
                'price' => 22.00,
                'image' => 'products/coaster-set.jpg',
                'stock' => 100,
            ],
            
            // Corporate
            [
                'category_id' => 5,
                'name' => json_encode(['bg' => 'Награда "Първенец"', 'en' => 'Award "Champion"', 'de' => 'Preis "Champion"']),
                'slug' => 'award-first',
                'description' => json_encode(['bg' => 'Персонална дървена награда за служители.', 'en' => 'Personalized wooden award for employees.', 'de' => 'Personalisierte Holzauszeichnung für Mitarbeiter.']),
                'price' => 40.00,
                'image' => 'products/award-1.jpg',
                'stock' => 30,
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
            'locale' => 'bg',
            'is_admin' => true,
            'password' => bcrypt('admin123'),
            'created_at' => now(),
            'updated_at' => now(),
        ]);

        // Demo Orders
        $orders = [
            ['name' => 'Maria Petrova', 'phone' => '+359888111222', 'address' => '15 Gradinska St', 'city' => 'Sofia', 'payment_method' => 'cash_on_delivery', 'subtotal' => 45.00, 'shipping' => 5.00, 'total' => 50.00, 'status' => 'processing', 'locale' => 'bg'],
            ['name' => 'Ivan Georgiev', 'phone' => '+359887333444', 'address' => '78 Vitosha Blvd', 'city' => 'Sofia', 'payment_method' => 'bank_transfer', 'subtotal' => 80.00, 'shipping' => 0.00, 'total' => 80.00, 'status' => 'confirmed', 'locale' => 'en'],
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
    git curl libpng-dev oniguruma-dev libxml2-dev zip unzip libsqlite3-dev sqlite3

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

## Step by Step

### Week 1: Foundation
- [ ] Laravel project with Docker
- [ ] Database migrations (SQLite)
- [ ] Multi-language setup (URL prefix)
- [ ] Models with JSON translations
- [ ] Install Filament + Breeze

### Week 2: Products
- [ ] Product listing (localized)
- [ ] Product page (localized)
- [ ] Seed with translations
- [ ] Language switcher

### Week 3: Cart + Checkout
- [ ] Cart (Livewire)
- [ ] Checkout form (localized)
- [ ] Payment: COD, Bank Transfer, PayPal
- [ ] Save to database with locale

### Week 4: Admin + Launch
- [ ] Order management in Filament
- [ ] Status updates
- [ ] Email confirmation
- [ ] **LAUNCH**

---

## Payment Methods

### 1. Cash on Delivery
- Popular in Bulgaria
- Customer pays to courier

### 2. Bank Transfer
- Account details shown after order

### 3. PayPal
- Link to PayPal.me

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
                Tables\Columns\TextColumn::make('locale')->label('Language'),
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

---

## Commands

```bash
# Build and run
docker-compose up -d --build

# Auto runs: migrations + seeds
# Access: http://localhost:8000
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
5. [ ] SEO per language
