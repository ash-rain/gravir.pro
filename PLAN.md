# Gravir.pro — Technical Plan

> Award-winning e-commerce for personalized wooden gifts. Built with modern PHP 8.5 & Laravel 12.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Tech Stack](#tech-stack)
3. [Modern PHP 8.5 Code](#modern-php-85-code)
4. [Database Schema](#database-schema)
5. [Multi-Language Strategy](#multi-language-strategy)
6. [Frontend Design](#frontend-design)
7. [Docker Setup](#docker-setup)
8. [Demo Seeder](#demo-seeder)
9. [Admin Panel](#admin-panel)
10. [Launch Checklist](#launch-checklist)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      gravir.pro                             │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Browser   │  │  Mobile    │  │    API    │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │
│         └────────────────┼────────────────┘                │
│                          ▼                                  │
│               ┌─────────────────────┐                       │
│               │   Laravel 12 App   │                       │
│               │   (Nginx/PHP 8.5) │                       │
│               └──────────┬──────────┘                       │
│                          │                                  │
│         ┌───────────────┼───────────────┐                 │
│         ▼               ▼               ▼                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐         │
│  │  Livewire  │  │  Filament  │  │   Queue   │         │
│  │  (Store)   │  │  (Admin)  │  │ (Emails)  │         │
│  └─────┬──────┘  └─────┬──────┘  └────────────┘         │
│        │                │                                  │
│        └────────┬─────┘                                  │
│                 ▼                                         │
│        ┌────────────────┐                                │
│        │    SQLite      │                                │
│        │  (database)    │                                │
│        └────────────────┘                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Version |
|-------|------------|---------|
| Framework | Laravel | 12.x |
| PHP | PHP | 8.5 |
| Frontend | Livewire | 4.x |
| Styling | Tailwind CSS | 4.x |
| Admin | Filament | 5.x |
| Database | SQLite | 3.x |
| Docker | Compose | 3.8 |

---

## Modern PHP 8.5 Code

### Enums for Type Safety

```php
// app/Enums/OrderStatus.php
<?php

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

    public function label(): string
    {
        return match ($this) {
            self::NEW => __('New'),
            self::CONFIRMED => __('Confirmed'),
            self::PROCESSING => __('Processing'),
            self::READY => __('Ready'),
            self::SHIPPED => __('Shipped'),
            self::DELIVERED => __('Delivered'),
            self::CANCELLED => __('Cancelled'),
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
// app/Enums/PaymentMethod.php
<?php

namespace App\Enums;

enum PaymentMethod: string
{
    case BANK_TRANSFER = 'bank_transfer';
    case CASH_ON_DELIVERY = 'cash_on_delivery';
    case PAYPAL = 'paypal';

    public function label(): string
    {
        return match ($this) {
            self::BANK_TRANSFER => __('Bank Transfer'),
            self::CASH_ON_DELIVERY => __('Cash on Delivery'),
            self::PAYPAL => __('PayPal'),
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

    public function instructions(): ?string
    {
        return match ($this) {
            self::BANK_TRANSFER => __('Bank: UniCredit Bulbank, IBAN: BG00UNCR70000000000000'),
            self::CASH_ON_DELIVERY => __('Pay to the courier upon delivery'),
            self::PAYPAL => __('You will receive PayPal payment link'),
            default => null,
        };
    }
}
```

### Readonly Models

```php
// app/Models/Product.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

final readonly class Product extends Model
{
    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }

    public function getFormattedPrice(): string
    {
        return number_format($this->price, 2) . ' €';
    }

    public function isInStock(): bool
    {
        return $this->stock > 0;
    }
}
```

### Attributes for Metadata

```php
// app/Attributes/SeoMetadata.php
<?php

namespace App\Attributes;

use Attribute;

#[Attribute]
readonly class SeoMetadata
{
    public function __construct(
        public string $title,
        public string $description,
        public array $keywords = [],
        public string $image = '/og-default.jpg',
    ) {}
}
```

### DTOs for Data Transfer

```php
// app/DTOs/CartItemData.php
<?php

namespace App\DTOs;

readonly class CartItemData
{
    public function __construct(
        public int $productId,
        public string $productName,
        public float $price,
        public int $quantity = 1,
        public ?string $customization = null,
    ) {}

    public function total(): float
    {
        return $this->price * $this->quantity;
    }

    public function toArray(): array
    {
        return [
            'product_id' => $this->productId,
            'product_name' => $this->productName,
            'price' => $this->price,
            'quantity' => $this->quantity,
            'customization' => $this->customization,
            'total' => $this->total(),
        ];
    }
}
```

### Service Classes

```php
// app/Services/CartService.php
<?php

namespace App\Services;

use App\DTOs\CartItemData;

class CartService
{
    private const CART_KEY = 'cart';

    public function get(): array
    {
        return session()->get(self::CART_KEY, []);
    }

    public function add(CartItemData $item): void
    {
        $cart = $this->get();
        
        $existingIndex = collect($cart)->search(
            fn ($i) => $i['product_id'] === $item->productId
        );

        if ($existingIndex !== false) {
            $cart[$existingIndex]['quantity'] += $item->quantity;
        } else {
            $cart[] = $item->toArray();
        }

        session()->put(self::CART_KEY, $cart);
    }

    public function remove(int $productId): void
    {
        $cart = collect($this->get())
            ->filter(fn ($item) => $item['product_id'] !== $productId)
            ->values()
            ->all();

        session()->put(self::CART_KEY, $cart);
    }

    public function clear(): void
    {
        session()->forget(self::CART_KEY);
    }

    public function subtotal(): float
    {
        return collect($this->get())
            ->sum(fn ($item) => $item['price'] * $item['quantity']);
    }

    public function shipping(): float
    {
        return $this->subtotal() >= 50 ? 0 : 5;
    }

    public function total(): float
    {
        return $this->subtotal() + $this->shipping();
    }

    public function count(): int
    {
        return collect($this->get())->sum('quantity');
    }
}
```

### Livewire Components (Modern Syntax)

```php
// app/Livewire/Cart/CartIndicator.php
<?php

namespace App\Livewire\Cart;

use Livewire\Component;
use Livewire\Attributes\On;
use App\Services\CartService;

final class CartIndicator extends Component
{
    public int $count = 0;

    public function mount(CartService $cart): void
    {
        $this->count = $cart->count();
    }

    #[On('cart-updated')]
    public function refresh(CartService $cart): void
    {
        $this->count = $cart->count();
    }

    public function render(): string
    {
        return <<<HTML
            <a href="/cart" class="relative p-2 text-gold-400 hover:text-gold-300 transition-colors">
                <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" 
                        d="M3 3h2l.4 2M7 13h10l4-8H5.4M7 13L5.4 5M7 13l-2.293 2.293c-.63.63-.184 1.707.707 1.707H17m0 0a2 2 0 100 4 2 2 0 000-4zm-8 2a2 2 0 11-4 0 2 2 0 014 0z" />
                </svg>
                @if(\$count > 0)
                    <span class="absolute -top-1 -right-1 bg-gold-500 text-charcoal-900 text-xs font-bold rounded-full w-5 h-5 flex items-center justify-center">
                        {{ \$count }}
                    </span>
                @endif
            </a>
        HTML;
    }
}
```

```php
// app/Livewire/Cart/Checkout.php
<?php

namespace App\Livewire\Cart;

use Livewire\Component;
use Livewire\Attributes\Validate;
use App\Services\CartService;
use App\Enums\PaymentMethod;
use App\Models\Order;
use App\Models\OrderItem;

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

    #[Validate('required')]
    public string $paymentMethod = PaymentMethod::CASH_ON_DELIVERY->value;

    public ?string $notes = null;

    public function submit(CartService $cart): \Illuminate\Http\RedirectResponse
    {
        $this->validate();

        if ($cart->count() === 0) {
            return redirect('/products');
        }

        $order = Order::create([
            'name' => $this->name,
            'phone' => $this->phone,
            'address' => $this->address,
            'city' => $this->city,
            'payment_method' => $this->paymentMethod,
            'subtotal' => $cart->subtotal(),
            'shipping' => $cart->shipping(),
            'total' => $cart->total(),
            'notes' => $this->notes,
            'locale' => app()->getLocale(),
            'status' => \App\Enums\OrderStatus::NEW->value,
        ]);

        foreach ($cart->get() as $item) {
            OrderItem::create([
                'order_id' => $order->id,
                'product_id' => $item['product_id'],
                'quantity' => $item['quantity'],
                'price' => $item['price'],
                'customization' => $item['customization'] ?? null,
            ]);
        }

        $cart->clear();

        return redirect("/thank-you?order={$order->id}");
    }

    public function render(CartService $cart): string
    {
        return <<<HTML
            <div class="max-w-2xl mx-auto px-6 py-12">
                <h1 class="font-serif text-4xl text-white mb-8">{{ __('Checkout') }}</h1>
                
                <form wire:submit="submit" class="space-y-6">
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Name') }}</label>
                        <input type="text" wire:model="name" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors" />
                        @error('name') <p class="text-red-400 text-sm mt-1">{{ \$message }}</p> @enderror
                    </div>
                    
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Phone') }}</label>
                        <input type="tel" wire:model="phone" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors" />
                        @error('phone') <p class="text-red-400 text-sm mt-1">{{ \$message }}</p> @enderror
                    </div>
                    
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Address') }}</label>
                        <input type="text" wire:model="address" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors" />
                        @error('address') <p class="text-red-400 text-sm mt-1">{{ \$message }}</p> @enderror
                    </div>
                    
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('City') }}</label>
                        <input type="text" wire:model="city" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors" />
                        @error('city') <p class="text-red-400 text-sm mt-1">{{ \$message }}</p> @enderror
                    </div>
                    
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Payment Method') }}</label>
                        <div class="space-y-3">
                            @foreach(\App\Enums\PaymentMethod::cases() as \$method)
                                <label class="flex items-center gap-3 p-4 bg-charcoal-800 border border-charcoal-700 rounded-lg cursor-pointer hover:border-gold-500 transition-colors">
                                    <input type="radio" wire:model="paymentMethod" value="{{ \$method->value }}" 
                                        class="text-gold-500 focus:ring-gold-500" />
                                    <span class="text-white">{{ \$method->label() }}</span>
                                </label>
                            @endforeach
                        </div>
                    </div>
                    
                    <div>
                        <label class="block text-gray-400 mb-2">{{ __('Notes') }}</label>
                        <textarea wire:model="notes" rows="3" 
                            class="w-full bg-charcoal-800 border border-charcoal-700 rounded-lg px-4 py-3 text-white focus:border-gold-500 focus:ring-1 focus:ring-gold-500 outline-none transition-colors"></textarea>
                    </div>
                    
                    <div class="bg-charcoal-800 rounded-lg p-6 border border-gold-500/20">
                        <div class="flex justify-between text-gray-400 mb-2">
                            <span>{{ __('Subtotal') }}</span>
                            <span>{{ number_format(\$cart->subtotal(), 2) }} €</span>
                        </div>
                        <div class="flex justify-between text-gray-400 mb-2">
                            <span>{{ __('Shipping') }}</span>
                            <span>{{ \$cart->shipping() === 0 ? __('Free') : number_format(\$cart->shipping(), 2) . ' €' }}</span>
                        </div>
                        <div class="flex justify-between text-white text-xl font-semibold pt-2 border-t border-charcoal-700">
                            <span>{{ __('Total') }}</span>
                            <span class="text-gold-400">{{ number_format(\$cart->total(), 2) }} €</span>
                        </div>
                    </div>
                    
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

## Database Schema

```php
// database/migrations/2026_01_01_000001_create_tables.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('phone', 20)->nullable();
            $table->text('address')->nullable();
            $table->string('city', 100)->nullable();
            $table->boolean('is_admin')->default(false);
            $table->string('locale')->default('bg');
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->rememberToken();
            $table->timestamps();
        });

        Schema::create('categories', function (Blueprint $table) {
            $table->id();
            $table->json('name'); // {"bg": "Сватбени", "en": "Wedding"}
            $table->string('slug')->unique();
            $table->json('description')->nullable();
            $table->string('image')->nullable();
            $table->timestamps();
        });

        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->foreignId('category_id')->constrained();
            $table->json('name');
            $table->string('slug')->unique();
            $table->json('description')->nullable();
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
            $table->string('payment_method');
            $table->decimal('subtotal', 10, 2);
            $table->decimal('shipping', 10, 2)->default(0);
            $table->decimal('total', 10, 2);
            $table->string('status')->default('new');
            $table->text('notes')->nullable();
            $table->string('tracking_number')->nullable();
            $table->string('locale')->default('bg');
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
    }
};
```

---

## Multi-Language Strategy

### Supported Languages

| Code | Language | Region |
|------|----------|--------|
| bg | Bulgarian | 🇧🇬 Default |
| en | English | 🌎 |
| de | German | 🇩🇪 |
| fr | French | 🇫🇷 |
| es | Spanish | 🇪🇸 |
| it | Italian | 🇮🇹 |
| ro | Romanian | 🇷🇴 |
| gr | Greek | 🇬🇷 |

### Language Routes

```php
// routes/web.php
<?php

use Illuminate\Support\Facades\Route;

Route::prefix('{locale}')->where(['locale' => 'bg|en|de|fr|es|it|ro|gr'])->group(function () {
    Route::get('/', \App\Livewire\Home::class)->name('home');
    Route::get('/products', \App\Livewire\Product\Index::class)->name('products');
    Route::get('/product/{slug}', \App\Livewire\Product\Show::class)->name('product');
    Route::get('/cart', \App\Livewire\Cart\Page::class)->name('cart');
    Route::get('/checkout', \App\Livewire\Cart\Checkout::class)->name('checkout');
    Route::get('/thank-you', \App\Livewire\Checkout\ThankYou::class)->name('thank-you');
});

// Redirect root to default locale
Route::get('/', fn () => redirect('/' . app()->getLocale()));
```

### Translation Helper

```php
// app/Helpers/Trans.php
<?php

use Illuminate\Support\Facades\Cache;

function trans_field(string $field, ?string $locale = null): string
{
    $locale ??= app()->getLocale();
    $data = json_decode($field, true);
    
    if (!$data) return $field;
    
    return $data[$locale] ?? $data['bg'] ?? $field;
}

function trans_name(string $field, ?string $locale = null): string
{
    return trans_field($field, $locale);
}
```

---

## Frontend Design

### Tailwind 4 Configuration

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
        'shimmer': 'shimmer 2s infinite',
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
        shimmer: {
          '0%': { backgroundPosition: '-200% 0' },
          '100%': { backgroundPosition: '200% 0' },
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

### Award-Winning UI Components

#### Product Card
```html
<!-- resources/views/components/product-card.blade.php -->
<article class="group relative bg-charcoal-800 rounded-2xl overflow-hidden border border-charcoal-700 hover:border-gold-500/30 transition-all duration-500 hover:shadow-2xl hover:shadow-gold-500/10">
    <a href="/{{ locale }}/product/{{ $product->slug }}" class="block">
        <div class="aspect-[4/5] bg-gradient-to-br from-charcoal-700 to-charcoal-800 relative overflow-hidden">
            <img src="/storage/{{ $product->image }}" 
                 alt="{{ trans_name($product->name) }}"
                 class="absolute inset-0 w-full h-full object-cover transition-transform duration-700 group-hover:scale-110"
                 loading="lazy" />
            <div class="absolute inset-0 bg-gradient-to-t from-charcoal-900/80 via-transparent to-transparent opacity-0 group-hover:opacity-100 transition-opacity duration-300"></div>
        </div>
        <div class="p-5">
            <h3 class="font-serif text-xl text-white mb-2 line-clamp-1 group-hover:text-gold-400 transition-colors">
                {{ trans_name($product->name) }}
            </h3>
            <p class="text-gray-400 text-sm line-clamp-2 mb-4">
                {{ trans_field($product->description) }}
            </p>
            <div class="flex items-center justify-between">
                <span class="text-gold-400 font-semibold">
                    {{ __('from') }} {{ number_format($product->price, 2) }} €
                </span>
                <span class="text-xs text-gray-500">
                    @if($product->stock > 0)
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

#### Hero Section
```html
<section class="relative min-h-screen flex items-center justify-center overflow-hidden">
    <!-- Animated Background -->
    <div class="absolute inset-0">
        <div class="absolute top-1/4 left-1/4 w-96 h-96 bg-gold-500/10 rounded-full blur-[128px] animate-pulse-gold"></div>
        <div class="absolute bottom-1/4 right-1/4 w-96 h-96 bg-gold-500/5 rounded-full blur-[128px]"></div>
    </div>
    
    <!-- Content -->
    <div class="relative z-10 text-center px-6 max-w-5xl mx-auto">
        <p class="text-gold-400 text-sm tracking-[0.3em] uppercase mb-6 animate-fade-in">
            {{ __('Personalized Wooden Gifts') }}
        </p>
        
        <h1 class="font-serif text-6xl md:text-8xl text-white mb-8 leading-tight animate-fade-in-up">
            {{ __('Gifts that') }}<br/>
            <span class="bg-gradient-to-r from-gold-300 via-gold-500 to-gold-300 bg-clip-text text-transparent">
                {{ __('tell stories') }}
            </span>
        </h1>
        
        <p class="text-xl text-gray-400 max-w-2xl mx-auto mb-12 animate-fade-in-up" style="animation-delay: 0.2s">
            {{ __('We engrave names, messages and photos onto wood. For moments that won\'t repeat.') }}
        </p>
        
        <div class="flex flex-col sm:flex-row gap-4 justify-center animate-fade-in-up" style="animation-delay: 0.4s">
            <a href="/{{ locale }}/products" 
                class="px-8 py-4 bg-gold-500 text-charcoal-900 font-semibold rounded-full hover:bg-gold-400 transition-all duration-300 hover:shadow-xl hover:shadow-gold-500/25 hover:-translate-y-1">
                {{ __('Explore Collection') }}
            </a>
            <a href="/{{ locale }}/contact" 
                class="px-8 py-4 border border-charcoal-600 text-white rounded-full hover:border-gold-500 hover:text-gold-400 transition-all duration-300">
                {{ __('Custom Orders') }}
            </a>
        </div>
    </div>
    
    <!-- Scroll Indicator -->
    <div class="absolute bottom-10 left-1/2 -translate-x-1/2 animate-float">
        <div class="w-6 h-10 border border-charcoal-600 rounded-full flex justify-center pt-2">
            <div class="w-1 h-2 bg-gold-500 rounded-full animate-pulse"></div>
        </div>
    </div>
</section>
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

```dockerfile
# Dockerfile
FROM php:8.5-cli

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    oniguruma-dev \
    libxml2-dev \
    zip \
    unzip \
    libsqlite3-dev \
    sqlite3 \
    nodejs \
    npm

# Install PHP extensions
RUN docker-php-ext-install pdo_sqlite mbstring exif pcntl

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# Copy application files
COPY . .

# Install dependencies
RUN composer install --no-dev --optimize-autoloader
RUN npm install && npm run build

# Expose port
EXPOSE 8000

# Start server
CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

---

## Demo Seeder

```php
// database/seeders/DatabaseSeeder.php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\DB;

final class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // Categories with all translations
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
                ]),
                'image' => 'categories/wedding.jpg',
            ],
            // ... more categories
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
            [
                'category_id' => 1,
                'name' => json_encode([
                    'bg' => 'Сватбена Табела "Заедно"',
                    'en' => 'Wedding Plaque "Together"',
                    'de' => 'Hochzeitsschild "Zusammen"',
                ]),
                'slug' => 'wedding-plaque-together',
                'description' => json_encode([
                    'bg' => 'Романтична сватбена табела с имената на двойката и датата.',
                    'en' => 'Romantic wedding plaque with couple names and date.',
                ]),
                'price' => 45.00,
                'image' => 'products/wedding-plaque-1.jpg',
                'stock' => 50,
            ],
            // ... more products
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
        foreach ([
            ['name' => 'Maria Petrova', 'phone' => '+359888111222', 'city' => 'Sofia', 'payment_method' => 'cash_on_delivery', 'total' => 50.00, 'status' => 'processing'],
            ['name' => 'Ivan Georgiev', 'phone' => '+359887333444', 'city' => 'Plovdiv', 'payment_method' => 'bank_transfer', 'total' => 80.00, 'status' => 'confirmed'],
        ] as $orderData) {
            $orderId = DB::table('orders')->insertGetId([
                ...$orderData,
                'address' => 'Sample Address',
                'subtotal' => $orderData['total'] - 5,
                'shipping' => 5,
                'locale' => 'bg',
                'created_at' => now(),
                'updated_at' => now(),
            ]);

            DB::table('order_items')->insert([
                'order_id' => $orderId,
                'product_id' => rand(1, 3),
                'quantity' => 1,
                'price' => $orderData['total'] - 5,
                'created_at' => now(),
                'updated_at' => now(),
            ]);
        }
    }
}
```

---

## Admin Panel

```php
// app/Filament/Resources/OrderResource.php
<?php

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
    protected static ?int $navigationSort = 1;

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('id')
                    ->label('#')
                    ->sortable()
                    ->toggleable(),
                
                Tables\Columns\TextColumn::make('name')
                    ->label('Customer')
                    ->searchable()
                    ->toggleable(),
                
                Tables\Columns\TextColumn::make('phone')
                    ->label('Phone')
                    ->toggleable(),
                
                Tables\Columns\TextColumn::make('city')
                    ->label('City')
                    ->toggleable(),
                
                Tables\Columns\TextColumn::make('total')
                    ->label('Total')
                    ->money('eur')
                    ->sortable()
                    ->toggleable(),
                
                Tables\Columns\TextColumn::make('payment_method')
                    ->label('Payment')
                    ->badge()
                    ->toggleable(),
                
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
                    ->sortable()
                    ->toggleable(),
                
                Tables\Columns\TextColumn::make('locale')
                    ->label('Lang')
                    ->toggleable(),
                
                Tables\Columns\TextColumn::make('created_at')
                    ->label('Date')
                    ->dateTime('d.m.Y H:i')
                    ->sortable()
                    ->toggleable(),
            ])
            ->defaultSort('created_at', 'desc')
            ->filters([
                Tables\Filters\SelectFilter::make('status')
                    ->options(collect(OrderStatus::cases())->mapWithKeys(fn ($s) => [$s->value => $s->label()])),
                Tables\Filters\SelectFilter::make('locale')
                    ->options(['bg' => 'Bulgarian', 'en' => 'English']),
            ])
            ->actions([
                Tables\Actions\ViewAction::make(),
                Tables\Actions\EditAction::make()
                    ->form([
                        Tables\Actions\EditAction\Field::make('status'),
                    ]),
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

## Launch Checklist

### Week 1: Foundation
- [ ] Laravel 12 project with Docker
- [ ] SQLite database + migrations
- [ ] Multi-language routing
- [ ] Filament admin panel
- [ ] Tailwind 4 configuration

### Week 2: Products
- [ ] Product listing page
- [ ] Product detail page
- [ ] Seed data with translations
- [ ] Language switcher

### Week 3: Store
- [ ] Cart functionality
- [ ] Checkout flow
- [ ] Payment methods
- [ ] Order confirmation

### Week 4: Polish & Launch
- [ ] Mobile responsiveness
- [ ] Performance optimization
- [ ] SEO metadata
- [ ] **🚀 LAUNCH**

---

## Post-Launch Features

1. [ ] User authentication (Breeze)
2. [ ] Customer reviews
3. [ ] Order tracking
4. [ ] Econt/Speedy integration
5. [ ] Newsletter signup
6. [ ] SEO per language
