# Gravir.pro — Complete Technical Specification

> Premium E-commerce Platform for Personalized Wooden Gifts  
> Laravel 12 + Livewire 4 + Tailwind 4 + PHP 8.5

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Technology Stack](#technology-stack)
3. [Architecture](#architecture)
4. [Database Schema](#database-schema)
5. [Models & Entities](#models--entities)
6. [Services & Business Logic](#services--business-logic)
7. [Livewire Components](#livewire-components)
8. [Frontend Design](#frontend-design)
9. [Multi-Language System](#multi-language-system)
10. [Payment Integration](#payment-integration)
11. [Admin Panel](#admin-panel)
12. [Docker Configuration](#docker-configuration)
13. [Development Workflow](#development-workflow)
14. [Seed Data](#seed-data)
15. [Security](#security)
16. [Performance](#performance)
17. [Deployment](#deployment)
18. [Roadmap](#roadmap)

---

## Project Overview

### Mission
Build an award-winning e-commerce platform for personalized wooden gifts, starting from Bulgaria and expanding across Europe.

### Goals
- Launch MVP in 4 weeks
- Multi-language support (BG default + 7 EU languages)
- Luxury design aesthetic
- Fast performance (Core Web Vitals)
- Scalable architecture

---

## Technology Stack

| Component | Technology | Version |
|-----------|------------|---------|
| Framework | Laravel | 12.x |
| PHP | PHP | 8.5 |
| Frontend | Livewire | 4.x |
| Styling | Tailwind CSS | 4.x |
| Admin | Filament | 5.x |
| Database | SQLite | 3.x |
| Docker | Compose | 3.8 |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                         gravir.pro                            │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│  │   Web UI    │   │  Mobile UI  │   │    API     │       │
│  │  (Livewire) │   │  (PWA)     │   │  (REST)    │       │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘       │
│         │                  │                  │               │
│         └──────────────────┼──────────────────┘               │
│                            ▼                                  │
│              ┌─────────────────────────┐                      │
│              │    Laravel 12 App      │                      │
│              │    (PHP 8.5 + Nginx)  │                      │
│              └───────────┬─────────────┘                      │
│                          │                                    │
│    ┌─────────────────────┼─────────────────────┐             │
│    ▼                     ▼                     ▼              │
│ ┌──────────┐      ┌──────────┐      ┌──────────┐          │
│ │ Livewire │      │ Filament │      │  Queue   │          │
│ │  Store   │      │  Admin   │      │ (Emails) │          │
│ └─────┬────┘      └─────┬────┘      └──────────┘          │
│       │                 │                                   │
│       └────────┬────────┘                                   │
│                ▼                                            │
│       ┌────────────────┐                                   │
│       │     SQLite     │                                   │
│       │   Database     │                                   │
│       └────────────────┘                                   │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Database Schema

### Migration File

```php
<?php
// database/migrations/2026_01_01_000001_create_gravir_tables.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        // Users Table
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('phone', 20)->nullable();
            $table->text('address')->nullable();
            $table->string('city', 100)->nullable();
            $table->string('locale', 10)->default('bg');
            $table->boolean('is_admin')->default(false);
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->rememberToken();
            $table->timestamps();
        });

        // Categories Table
        Schema::create('categories', function (Blueprint $table) {
            $table->id();
            $table->json('name'); // {"bg": "Сватбени", "en": "Wedding"}
            $table->string('slug')->unique();
            $table->json('description')->nullable();
            $table->string('image')->nullable();
            $table->integer('sort_order')->default(0);
            $table->boolean('is_active')->default(true);
            $table->timestamps();
        });

        // Products Table
        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->foreignId('category_id')->constrained('categories')->cascadeOnDelete();
            $table->json('name');
            $table->string('slug')->unique();
            $table->json('description')->nullable();
            $table->decimal('price', 10, 2);
            $table->string('image');
            $table->json('images')->nullable(); // Gallery
            $table->boolean('is_active')->default(true);
            $table->integer('stock')->default(100);
            $table->integer('views')->default(0);
            $table->integer('sales')->default(0);
            $table->timestamps();
        });

        // Orders Table
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->nullable()->constrained('users')->nullOnDelete();
            $table->string('order_number', 20)->unique();
            $table->string('name');
            $table->string('phone', 20);
            $table->text('address');
            $table->string('city', 100);
            $table->string('payment_method', 50);
            $table->string('payment_status', 20)->default('pending');
            $table->decimal('subtotal', 10, 2);
            $table->decimal('shipping', 10, 2)->default(0);
            $table->decimal('discount', 10, 2)->default(0);
            $table->decimal('total', 10, 2);
            $table->string('status', 20)->default('new');
            $table->text('notes')->nullable();
            $table->string('tracking_number', 50)->nullable();
            $table->string('locale', 10)->default('bg');
            $table->timestamp('paid_at')->nullable();
            $table->timestamp('shipped_at')->nullable();
            $table->timestamps();
        });

        // Order Items Table
        Schema::create('order_items', function (Blueprint $table) {
            $table->id();
            $table->foreignId('order_id')->constrained('orders')->cascadeOnDelete();
            $table->foreignId('product_id')->constrained('products')->cascadeOnDelete();
            $table->string('product_name'); // Snapshot
            $table->decimal('product_price', 10, 2); // Snapshot
            $table->integer('quantity')->default(1);
            $table->json('customization')->nullable(); // {"text": "John", "font": "classic"}
            $table->timestamps();
        });

        // Pages Table (for CMS)
        Schema::create('pages', function (Blueprint $table) {
            $table->id();
            $table->string('slug')->unique();
            $table->json('title');
            $table->json('content')->nullable();
            $table->boolean('is_published')->default(false);
            $table->timestamps();
        });

        // Subscribers Table (Newsletter)
        Schema::create('subscribers', function (Blueprint $table) {
            $table->id();
            $table->string('email')->unique();
            $table->string('locale', 10)->default('bg');
            $table->boolean('is_active')->default(true);
            $table->timestamp('subscribed_at')->useCurrent();
            $table->timestamps();
        });
    }
};
```

---

## Models & Entities

### Enums

```php
<?php
// app/Enums/OrderStatus.php

namespace App\Enums;

enum OrderStatus: string
{
    case NEW = 'new';
    case CONFIRMED = 'confirmed';
    case PROCESSING = 'processing';
    case READY = 'ready';
    case SHIPPED = 'shipped';
    case DELIVERED = 'delivered';
    case CANCELLED = 'cancelled';

    public function label(string $locale = 'bg'): string
    {
        return match ($this) {
            self::NEW => __('status.new', locale: $locale),
            self::CONFIRMED => __('status.confirmed', locale: $locale),
            self::PROCESSING => __('status.processing', locale: $locale),
            self::READY => __('status.ready', locale: $locale),
            self::SHIPPED => __('status.shipped', locale: $locale),
            self::DELIVERED => __('status.delivered', locale: $locale),
            self::CANCELLED => __('status.cancelled', locale: $locale),
        };
    }

    public function color(): string
    {
        return match ($this) {
            self::NEW => 'gray',
            self::CONFIRMED => 'blue',
            self::PROCESSING => 'yellow',
            self::READY => 'orange',
            self::SHIPPED => 'indigo',
            self::DELIVERED => 'green',
            self::CANCELLED => 'red',
        };
    }

    public function canTransitionTo(self $target): bool
    {
        return match ($this) {
            self::NEW => in_array($target, [self::CONFIRMED, self::CANCELLED]),
            self::CONFIRMED => in_array($target, [self::PROCESSING, self::CANCELLED]),
            self::PROCESSING => in_array($target, [self::READY]),
            self::READY => in_array($target, [self::SHIPPED]),
            self::SHIPPED => in_array($target, [self::DELIVERED]),
            default => false,
        };
    }
}
```

```php
<?php
// app/Enums/PaymentMethod.php

namespace App\Enums;

enum PaymentMethod: string
{
    case BANK_TRANSFER = 'bank_transfer';
    case CASH_ON_DELIVERY = 'cash_on_delivery';
    case PAYPAL = 'paypal';

    public function label(string $locale = 'bg'): string
    {
        return match ($this) {
            self::BANK_TRANSFER => __('payment.bank_transfer', locale: $locale),
            self::CASH_ON_DELIVERY => __('payment.cash_on_delivery', locale: $locale),
            self::PAYPAL => __('payment.paypal', locale: $locale),
        };
    }

    public function icon(): string
    {
        return match ($this) {
            self::BANK_TRANSFER => 'heroicon-o-building-library',
            self::CASH_ON_DELIVERY => 'heroicon-o-currency-dollar',
            self::PAYPAL => 'heroicon-o-credit-card',
        };
    }
}
```

### Models

```php
<?php
// app/Models/Product.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

final readonly class Product extends Model
{
    protected $fillable = [
        'category_id',
        'name',
        'slug', 
        'description',
        'price',
        'image',
        'images',
        'is_active',
        'stock',
        'views',
        'sales',
    ];

    protected $casts = [
        'price' => 'decimal:2',
        'images' => 'array',
        'is_active' => 'boolean',
        'stock' => 'integer',
        'views' => 'integer',
        'sales' => 'integer',
    ];

    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }

    public function orderItems(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    public function getLocalizedName(string $locale = 'bg'): string
    {
        $names = json_decode($this->name, true);
        return $names[$locale] ?? $names['bg'] ?? $this->name;
    }

    public function getFormattedPrice(): string
    {
        return number_format($this->price, 2) . ' €';
    }

    public function isInStock(): bool
    {
        return $this->stock > 0;
    }

    public function isOnSale(): bool
    {
        return false; // Future: sale_price field
    }
}
```

```php
<?php
// app/Models/Order.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Order extends Model
{
    protected $fillable = [
        'user_id',
        'order_number',
        'name',
        'phone',
        'address',
        'city',
        'payment_method',
        'payment_status',
        'subtotal',
        'shipping',
        'discount',
        'total',
        'status',
        'notes',
        'tracking_number',
        'locale',
        'paid_at',
        'shipped_at',
    ];

    protected $casts = [
        'subtotal' => 'decimal:2',
        'shipping' => 'decimal:2',
        'discount' => 'decimal:2',
        'total' => 'decimal:2',
        'paid_at' => 'datetime',
        'shipped_at' => 'datetime',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    public static function generateOrderNumber(): string
    {
        return 'GVR-' . date('Ymd') . '-' . strtoupper(uniqid());
    }

    public function getPaymentMethodLabel(string $locale = 'bg'): string
    {
        return match ($this->payment_method) {
            'bank_transfer' => __('payment.bank_transfer', locale: $locale),
            'cash_on_delivery' => __('payment.cash_on_delivery', locale: $locale),
            'paypal' => __('payment.paypal', locale: $locale),
            default => $this->payment_method,
        };
    }
}
```

---

## Services & Business Logic

### Cart Service

```php
<?php
// app/Services/CartService.php

namespace App\Services;

use Illuminate\Support\Collection;

class CartService
{
    private const CART_KEY = 'cart';
    private const SHIPPING_THRESHOLD = 50;
    private const SHIPPING_COST = 5;

    public function get(): Collection
    {
        return collect(session()->get(self::CART_KEY, []));
    }

    public function add(int $productId, string $productName, float $price, int $quantity = 1, ?array $customization = null): void
    {
        $cart = $this->get();
        
        $index = $cart->search(fn ($item) => $item['product_id'] === $productId && $item['customization'] === $customization);

        if ($index !== false) {
            $item = $cart->get($index);
            $item['quantity'] += $quantity;
            $cart->put($index, $item);
        } else {
            $cart->push([
                'product_id' => $productId,
                'product_name' => $productName,
                'price' => $price,
                'quantity' => $quantity,
                'customization' => $customization,
            ]);
        }

        session()->put(self::CART_KEY, $cart->values()->all());
    }

    public function updateQuantity(int $productId, int $quantity): void
    {
        $cart = $this->get()->map(function ($item) use ($productId, $quantity) {
            if ($item['product_id'] === $productId) {
                $item['quantity'] = max(1, $quantity);
            }
            return $item;
        });

        session()->put(self::CART_KEY, $cart->all());
    }

    public function remove(int $productId): void
    {
        $cart = $this->get()->filter(fn ($item) => $item['product_id'] !== $productId);
        session()->put(self::CART_KEY, $cart->values()->all());
    }

    public function clear(): void
    {
        session()->forget(self::CART_KEY);
    }

    public function subtotal(): float
    {
        return $this->get()->sum(fn ($item) => $item['price'] * $item['quantity']);
    }

    public function shipping(): float
    {
        return $this->subtotal() >= self::SHIPPING_THRESHOLD ? 0 : self::SHIPPING_COST;
    }

    public function discount(): float
    {
        return 0; // Future: coupon codes
    }

    public function total(): float
    {
        return $this->subtotal() + $this->shipping() - $this->discount();
    }

    public function count(): int
    {
        return $this->get()->sum('quantity');
    }

    public function isEmpty(): bool
    {
        return $this->get()->isEmpty();
    }
}
```

### Order Service

```php
<?php
// app/Services/OrderService.php

namespace App\Services;

use App\Models\Order;
use App\Models\OrderItem;
use App\Enums\OrderStatus;
use Illuminate\Support\Facades\DB;

class OrderService
{
    public function createFromCart(CartService $cart, array $customerData, string $paymentMethod, string $locale = 'bg'): Order
    {
        return DB::transaction(function () use ($cart, $customerData, $paymentMethod, $locale) {
            $order = Order::create([
                'order_number' => Order::generateOrderNumber(),
                'name' => $customerData['name'],
                'phone' => $customerData['phone'],
                'address' => $customerData['address'],
                'city' => $customerData['city'],
                'payment_method' => $paymentMethod,
                'payment_status' => $paymentMethod === 'cash_on_delivery' ? 'pending' : 'pending',
                'subtotal' => $cart->subtotal(),
                'shipping' => $cart->shipping(),
                'discount' => $cart->discount(),
                'total' => $cart->total(),
                'status' => OrderStatus::NEW->value,
                'notes' => $customerData['notes'] ?? null,
                'locale' => $locale,
            ]);

            foreach ($cart->get() as $item) {
                OrderItem::create([
                    'order_id' => $order->id,
                    'product_id' => $item['product_id'],
                    'product_name' => $item['product_name'],
                    'product_price' => $item['price'],
                    'quantity' => $item['quantity'],
                    'customization' => $item['customization'] ?? null,
                ]);
            }

            $cart->clear();

            return $order;
        });
    }

    public function updateStatus(Order $order, OrderStatus $newStatus): bool
    {
        if (!$order->status->canTransitionTo($newStatus)) {
            return false;
        }

        $order->update(['status' => $newStatus->value]);

        if ($newStatus === OrderStatus::SHIPPED) {
            $order->update(['shipped_at' => now()]);
        }

        return true;
    }
}
```

---

## Livewire Components

### Home Page

```php
<?php
// app/Livewire/Home.php

namespace App\Livewire;

use Livewire\Component;
use App\Models\Product;
use App\Models\Category;

final class Home extends Component
{
    public array $featuredProducts = [];
    public array $categories = [];

    public function mount(): void
    {
        $this->featuredProducts = Product::active()
            ->inRandomOrder()
            ->limit(8)
            ->get()
            ->map(fn ($p) => [
                'id' => $p->id,
                'slug' => $p->slug,
                'name' => $p->getLocalizedName(app()->getLocale()),
                'description' => strip_tags($p->getLocalizedDescription(app()->getLocale())),
                'price' => $p->formatted_price,
                'image' => $p->image,
                'in_stock' => $p->isInStock(),
            ])
            ->all();

        $this->categories = Category::active()
            ->ordered()
            ->limit(5)
            ->get()
            ->map(fn ($c) => [
                'slug' => $c->slug,
                'name' => $c->getLocalizedName(app()->getLocale()),
                'image' => $c->image,
            ])
            ->all();
    }

    public function render(): string
    {
        return <<<'HTML'
            <div>
                <!-- Hero Section -->
                <section class="relative min-h-screen flex items-center justify-center overflow-hidden">
                    <div class="absolute inset-0">
                        <div class="absolute top-1/4 left-1/4 w-96 h-96 bg-gold-500/10 rounded-full blur-[128px] animate-pulse"></div>
                        <div class="absolute bottom-1/4 right-1/4 w-96 h-96 bg-gold-500/5 rounded-full blur-[128px]"></div>
                    </div>
                    
                    <div class="relative z-10 text-center px-6 max-w-5xl mx-auto">
                        <p class="text-gold-400 text-sm tracking-[0.3em] uppercase mb-6 animate-fade-in">
                            {{ __("Personalized Wooden Gifts") }}
                        </p>
                        
                        <h1 class="font-serif text-6xl md:text-8xl text-white mb-8 leading-tight animate-fade-in-up">
                            {{ __("Gifts that") }}<br/>
                            <span class="bg-gradient-to-r from-gold-300 via-gold-500 to-gold-300 bg-clip-text text-transparent">
                                {{ __("tell stories") }}
                            </span>
                        </h1>
                        
                        <p class="text-xl text-gray-400 max-w-2xl mx-auto mb-12">
                            {{ __("We engrave names, messages and photos onto wood. For moments that won't repeat.") }}
                        </p>
                        
                        <div class="flex flex-col sm:flex-row gap-4 justify-center">
                            <a href="/{{ locale }}/products" 
                                class="px-8 py-4 bg-gold-500 text-charcoal-900 font-semibold rounded-full hover:bg-gold-400 transition-all">
                                {{ __("Explore Collection") }}
                            </a>
                        </div>
                    </div>
                </section>

                <!-- Categories -->
                <section class="py-20 bg-charcoal-900">
                    <div class="max-w-7xl mx-auto px-6">
                        <h2 class="font-serif text-4xl text-white mb-12 text-center">{{ __("Our Collections") }}</h2>
                        
                        <div class="grid md:grid-cols-3 lg:grid-cols-5 gap-6">
                            @foreach($categories as $category)
                                <a href="/{{ locale }}/products?category={{ $category['slug'] }}" 
                                   class="group relative aspect-square rounded-2xl overflow-hidden">
                                    <img src="/storage/{{ $category['image'] }}" 
                                         alt="{{ $category['name'] }}"
                                         class="absolute inset-0 w-full h-full object-cover group-hover:scale-110 transition-transform duration-500" />
                                    <div class="absolute inset-0 bg-gradient-to-t from-charcoal-900/90 to-transparent"></div>
                                    <div class="absolute bottom-4 left-4 right-4">
                                        <h3 class="text-white font-serif text-xl group-hover:text-gold-400 transition-colors">
                                            {{ $category['name'] }}
                                        </h3>
                                    </div>
                                </a>
                            @endforeach
                        </div>
                    </div>
                </section>

                <!-- Featured Products -->
                <section class="py-20 bg-charcoal-800">
                    <div class="max-w-7xl mx-auto px-6">
                        <h2 class="font-serif text-4xl text-white mb-12 text-center">{{ __("Featured Products") }}</h2>
                        
                        <div class="grid md:grid-cols-2 lg:grid-cols-4 gap-6">
                            @foreach($featuredProducts as $product)
                                <x-product-card :product="$product" />
                            @endforeach
                        </div>
                    </div>
                </section>
            </div>
        HTML;
    }
}
```

### Checkout Component

```php
<?php
// app/Livewire/Cart/Checkout.php

namespace App\Livewire\Cart;

use Livewire\Component;
use Livewire\Attributes\Validate;
use App\Services\CartService;
use App\Services\OrderService;
use App\Enums\PaymentMethod;

final class Checkout extends Component
{
    #[Validate('required|min:2')]
    public string $name = '';

    #[Validate('required|min:10')]
    public string $phone = '';

    #[Validate('required')]
    public string $address = '';

    #[Validate('required')]
    public string $city = '';

    #[Validate('required|in:bank_transfer,cash_on_delivery,paypal')]
    public string $paymentMethod = PaymentMethod::CASH_ON_DELIVERY->value;

    public ?string $notes = null;

    public function submit(CartService $cart, OrderService $orderService): \Illuminate\Http\RedirectResponse
    {
        $this->validate();

        if ($cart->isEmpty()) {
            return redirect('/' . app()->getLocale() . '/products');
        }

        $order = $orderService->createFromCart(
            $cart,
            [
                'name' => $this->name,
                'phone' => $this->phone,
                'address' => $this->address,
                'city' => $this->city,
                'notes' => $this->notes,
            ],
            $this->paymentMethod,
            app()->getLocale()
        );

        return redirect('/' . app()->getLocale() . '/thank-you?order=' . $order->order_number);
    }

    public function render(CartService $cart): string
    {
        return <<<'HTML'
            <div class="max-w-2xl mx-auto px-6 py-12">
                <h1 class="font-serif text-4xl text-white mb-8">{{ __('Checkout') }}</h1>
                
                <form wire:submit="submit" class="space-y-6">
                    <!-- Name -->
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Name') }}</label>
                        <input type="text" wire:model="name" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors" />
                        @error('name') <p class="text-red-400 text-sm mt-1">{{ $message }}</p> @enderror
                    </div>
                    
                    <!-- Phone -->
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Phone') }}</label>
                        <input type="tel" wire:model="phone" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors" />
                        @error('phone') <p class="text-red-400 text-sm mt-1">{{ $message }}</p> @enderror
                    </div>
                    
                    <!-- Address -->
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Address') }}</label>
                        <input type="text" wire:model="address" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors" />
                        @error('address') <p class="text-red-400 text-sm mt-1">{{ $message }}</p> @enderror
                    </div>
                    
                    <!-- City -->
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('City') }}</label>
                        <input type="text" wire:model="city" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors" />
                        @error('city') <p class="text-red-400 text-sm mt-1">{{ $message }}</p> @enderror
                    </div>
                    
                    <!-- Payment Method -->
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Payment Method') }}</label>
                        <div class="space-y-3">
                            @foreach(PaymentMethod::cases() as $method)
                                <label class="flex items-center gap-3 p-4 bg-charcoal-800 border border-charcoal-700 rounded-lg cursor-pointer hover:border-gold-500 transition-colors">
                                    <input type="radio" wire:model="paymentMethod" value="{{ $method->value }}" 
                                        class="text-gold-500 focus:ring-gold-500" />
                                    <span class="text-white">{{ $method->label() }}</span>
                                </label>
                            @endforeach
                        </div>
                    </div>
                    
                    <!-- Notes -->
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Notes') }}</label>
                        <textarea wire:model="notes" rows="3" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors"></textarea>
                    </div>
                    
                    <!-- Order Summary -->
                    <div class="bg-charcoal-800 rounded-lg p-6 border border-gold-500/20">
                        <div class="flex justify-between text-gray-400 mb-2">
                            <span>{{ __('Subtotal') }}</span>
                            <span>{{ number_format($cart->subtotal(), 2) }} €</span>
                        </div>
                        <div class="flex justify-between text-gray-400 mb-2">
                            <span>{{ __('Shipping') }}</span>
                            <span>{{ $cart->shipping() === 0 ? __('Free') : number_format($cart->shipping(), 2) . ' €' }}</span>
                        </div>
                        <div class="flex justify-between text-white text-xl font-semibold pt-2 border-t border-charcoal-700">
                            <span>{{ __('Total') }}</span>
                            <span class="text-gold-400">{{ number_format($cart->total(), 2) }} €</span>
                        </div>
                    </div>
                    
                    <!-- Submit -->
                    <button type="submit" wire:loading.attr="disabled"
                        class="w-full bg-gold-500 text-charcoal-900 font-semibold py-4 rounded-lg hover:bg-gold-400 transition-colors disabled:opacity-50">
                        <span wire:loading.remove>{{ __('Place Order') }}</span>
                        <span wire:loading>{{ __('Processing...') }}</span>
                    </button>
                </form>
            </div>
        HTML;
    }
}
```

---

## Frontend Design

### Tailwind Configuration

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    "./resources/**/*.blade.php",
    "./vendor/livewire/**/*.*",
  ],
  theme: {
    extend: {
      colors: {
        charcoal: {
          950: '#050505',
          900: '#0a0a0a',
          800: '#141414',
          700: '#1f1f1f',
          600: '#2a2a2a',
          500: '#3a3a3a',
        },
        gold: {
          50: '#faf8f3',
          100: '#f5f0e3',
          200: '#e8dcc4',
          300: '#d9c49e',
          400: '#c9a96f',
          500: '#b8954a',
          600: '#a67c3d',
          700: '#8a6333',
        },
      },
      fontFamily: {
        serif: ['Cormorant Garamond', 'Georgia', 'serif'],
        sans: ['Inter', 'system-ui', 'sans-serif'],
      },
      animation: {
        'fade-in': 'fadeIn 0.5s ease-out',
        'fade-in-up': 'fadeInUp 0.6s ease-out',
        'float': 'float 6s ease-in-out infinite',
        'pulse-gold': 'pulseGold 2s infinite',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        fadeInUp: {
          '0%': { opacity: '0', transform: 'translateY(20px)' },
          '100%': { opacity: '1', transform: 'translateY(0)' },
        },
        float: {
          '0%, 100%': { transform: 'translateY(0)' },
          '50%': { transform: 'translateY(-10px)' },
        },
        pulseGold: {
          '0%, 100%': { opacity: '1' },
          '50%': { opacity: '0.6' },
        },
      },
    },
  },
  plugins: [],
}
```

### Product Card Component

```html
<!-- resources/views/components/product-card.blade.php -->
@props(['product'])

<article class="group relative bg-charcoal-800 rounded-2xl overflow-hidden border border-charcoal-700 hover:border-gold-500/30 transition-all duration-500 hover:shadow-2xl hover:shadow-gold-500/10">
    <a href="/{{ locale }}/product/{{ $product['slug'] }}" class="block">
        <div class="aspect-[4/5] bg-gradient-to-br from-charcoal-700 to-charcoal-800 relative overflow-hidden">
            <img src="/storage/{{ $product['image'] }}" 
                 alt="{{ $product['name'] }}"
                 class="absolute inset-0 w-full h-full object-cover transition-transform duration-700 group-hover:scale-110"
                 loading="lazy" />
            <div class="absolute inset-0 bg-gradient-to-t from-charcoal-900/80 via-transparent to-transparent opacity-0 group-hover:opacity-100 transition-opacity duration-300"></div>
        </div>
        <div class="p-5">
            <h3 class="font-serif text-xl text-white mb-2 line-clamp-1 group-hover:text-gold-400 transition-colors">
                {{ $product['name'] }}
            </h3>
            <p class="text-gray-400 text-sm line-clamp-2 mb-4">
                {{ $product['description'] }}
            </p>
            <div class="flex items-center justify-between">
                <span class="text-gold-400 font-semibold">
                    {{ __('from') }} {{ $product['price'] }}
                </span>
                <span class="text-xs text-gray-500">
                    @if($product['in_stock'])
                        <span class="text-green-400">●</span> {{ __('In Stock') }}
                    @else
                        <span class="text-red-400">●</span> {{ __('Out of Stock') }}
                    @endif
                </span>
            </div>
        </div>
    </a>
</article>
```

---

## Multi-Language System

### Supported Languages

| Code | Language | Flag |
|------|----------|------|
| bg | Bulgarian | 🇧🇬 |
| en | English | 🇬🇧 |
| de | German | 🇩🇪 |
| fr | French | 🇫🇷 |
| es | Spanish | 🇪🇸 |
| it | Italian | 🇮🇹 |
| ro | Romanian | 🇷🇴 |
| gr | Greek | 🇬🇷 |

### Route Configuration

```php
// routes/web.php

use Illuminate\Support\Facades\Route;

Route::prefix('{locale}')
    ->where(['locale' => 'bg|en|de|fr|es|it|ro|gr'])
    ->group(function () {
        Route::get('/', \App\Livewire\Home::class)->name('home');
        Route::get('/products', \App\Livewire\Product\Index::class)->name('products');
        Route::get('/product/{slug}', \App\Livewire\Product\Show::class)->name('product.show');
        Route::get('/cart', \App\Livewire\Cart\Page::class)->name('cart');
        Route::get('/checkout', \App\Livewire\Cart\Checkout::class)->name('checkout');
        Route::get('/thank-you', \App\Livewire\Checkout\ThankYou::class)->name('checkout.thank-you');
    });

Route::get('/', fn () => redirect('/' . app()->getLocale()));
```

### Translation Helper

```php
// app/Helpers/Trans.php

function trans_field(string $field, ?string $locale = null): string
{
    $locale ??= app()->getLocale();
    $data = json_decode($field, true);
    
    if (!$data || !is_array($data)) {
        return $field;
    }
    
    return $data[$locale] ?? $data['bg'] ?? $field;
}
```

---

## Payment Integration

### Payment Methods

1. **Cash on Delivery** - Pay to courier on delivery
2. **Bank Transfer** - Direct bank transfer
3. **PayPal** - PayPal.me link

```php
// In Order Model
public function getPaymentInstructionsAttribute(): ?string
{
    return match ($this->payment_method) {
        'bank_transfer' => "Bank: UniCredit Bulbank\nIBAN: BG00UNCR70000000000000\nRecipient: Gravir.pro Ltd",
        'cash_on_delivery' => "Pay " . number_format($this->total, 2) . " € to the courier upon delivery.",
        'paypal' => "Pay via PayPal: paypal.me/gravirpro",
        default => null,
    };
}
```

---

## Admin Panel

### Filament Order Resource

```php
<?php
// app/Filament/Resources/OrderResource.php

namespace App\Filament\Resources;

use Filament\Tables;
use Filament\Tables\Table;
use Filament\Resources\Resource;
use App\Models\Order;
use App\Enums\OrderStatus;

final class OrderResource extends Resource
{
    protected static ?string $model = Order::class;
    protected static ?string $navigationIcon = 'heroicon-o-shopping-cart';
    protected static ?string $navigationLabel = 'Orders';

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('order_number')
                    ->label('Order #')
                    ->searchable()
                    ->sortable(),
                
                Tables\Columns\TextColumn::make('name')
                    ->label('Customer')
                    ->searchable(),
                
                Tables\Columns\TextColumn::make('phone')
                    ->label('Phone'),
                
                Tables\Columns\TextColumn::make('city')
                    ->label('City'),
                
                Tables\Columns\TextColumn::make('total')
                    ->label('Total')
                    ->money('eur')
                    ->sortable(),
                
                Tables\Columns\BadgeColumn::make('payment_method')
                    ->label('Payment')
                    ->colors(['blue']),
                
                Tables\Columns\BadgeColumn::make('status')
                    ->label('Status')
                    ->colors([
                        'gray' => 'new',
                        'blue' => 'confirmed',
                        'yellow' => 'processing',
                        'orange' => 'ready',
                        'indigo' => 'shipped',
                        'green' => 'delivered',
                        'red' => 'cancelled',
                    ])
                    ->sortable(),
                
                Tables\Columns\TextColumn::make('locale')
                    ->label('Lang')
                    ->toggleable(),
                
                Tables\Columns\TextColumn::make('created_at')
                    ->label('Date')
                    ->dateTime('d.m.Y H:i')
                    ->sortable(),
            ])
            ->defaultSort('created_at', 'desc')
            ->filters([
                Tables\Filters\SelectFilter::make('status')
                    ->options(collect(OrderStatus::cases())->mapWithKeys(fn ($s) => [$s->value => $s->label()])),
            ])
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

## Docker Configuration

### docker-compose.yml

```yaml
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
      - APP_LOCALE=bg
      - APP_FALLBACK_LOCALE=bg
    working_dir: /var/www/html
    command: >
      sh -c "
        echo '⚡ Starting Gravir.pro...' &&
        touch database/database.sqlite &&
        php artisan migrate --force &&
        php artisan db:seed --force &&
        php artisan serve --host=0.0.0.0 --port=8000
      "
```

### Dockerfile

```dockerfile
FROM php:8.5-cli

RUN apt-get update && apt-get install -y \
    git curl libpng-dev oniguruma-dev libxml2-dev zip unzip libsqlite3-dev sqlite3 nodejs npm

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

## Development Workflow

```bash
# Install dependencies
docker-compose build

# Start development server
docker-compose up -d

# Run migrations
docker-compose exec app php artisan migrate

# Seed database
docker-compose exec app php artisan db:seed

# View logs
docker-compose logs -f app

# Access the application
# http://localhost:8000

# Access admin panel
# http://localhost:8000/admin
# Email: admin@gravir.pro
# Password: admin123
```

---

## Seed Data

### Categories
1. Wedding Gifts (Сватбени Подаръци)
2. Name Plates (Именни Табели)
3. Photo Engravings (Фотогравировки)
4. Home Decor (Домашен Декор)
5. Corporate (Корпоративни)

### Sample Products (10 total)
- Wedding Plaque "Together" - €45
- Wooden Ring Box - €35
- Wedding Cutting Board - €40
- Name Plate Classic - €18
- Name Plate Modern - €22
- Photo Engraving Memory - €45
- Pet Portrait - €35
- Cutting Board Chef - €35
- Coaster Set 4pc - €22
- Award Champion - €40

### Admin User
- Email: admin@gravir.pro
- Password: admin123

### Demo Orders (3)
1. Maria Petrova - Sofia - COD - €50 - Processing
2. Ivan Georgiev - Sofia - Bank Transfer - €80 - Confirmed

---

## Security

- CSRF protection (Laravel built-in)
- XSS sanitization (Blade)
- SQL injection prevention (Eloquent)
- Rate limiting on forms
- Secure session handling
- File upload validation
- HTTPS enforced in production

---

## Performance

- SQLite for fast queries
- Lazy loading images
- Cached translations
- Optimized Tailwind CSS
- CDN-ready static assets

---

## Deployment

### Production Checklist
- [ ] Set APP_ENV=production
- [ ] Configure HTTPS (Let's Encrypt)
- [ ] Set secure session config
- [ ] Configure logging
- [ ] Set up backups
- [ ] Configure queue workers

---

## Roadmap

### Phase 1 (Week 1-4): MVP
- [ ] Project setup
- [ ] Database & models
- [ ] Products & categories
- [ ] Cart & checkout
- [ ] Admin panel
- [ ] Multi-language
- [ ] Launch

### Phase 2 (Month 2-3): Polish
- [ ] User accounts
- [ ] Reviews
- [ ] Order tracking
- [ ] SEO optimization

### Phase 3 (Month 4-6): Scale
- [ ] Payment integration (Stripe)
- [ ] Courier integration
- [ ] Analytics
- [ ] Marketing tools
