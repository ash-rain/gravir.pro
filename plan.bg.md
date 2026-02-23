# Gravir.pro — Технически План

## Само Най-Важното

Правим онлайн магазин за персонализирани дървени подаръци. Колкото по-просто, толкова по-добре.

---

## Какво Ни Трябва

### Задължително
1. Продукти с снимки и цени
2. Кошница
3. Поръчка (име, телефон, адрес)
4. Плащане (банков превод)
5. Потвърждение по имейл

### Кво Не Ни Трябва Сега
- Потребителски профили
- Отзиви
- Проследяване на поръчки
- Многоезична поддръжка
- Сложна кутия за дизайн

---

## Страници

```
/                  → Начало
/products          → Продукти
/product/{slug}   → Детайли за продукт
/cart              → Кошница
/checkout          → Поръчка
/thank-you         → Благодаря
/admin             → Админ панел
/admin/orders      → Поръчки
```

---

## База Данни

```
users              → Име, имейл, телефон, админ (да/не)
categories         → Име, опис, изображение
products           → Име, опис, цена, изображение, category_id
orders             → user_id, name, phone, address, total, status, notes
order_items        → order_id, product_id, quantity, price
```

---

## Технологии

```bash
# Инсталация
composer create-project laravel/laravel gravir
cd gravir
composer require livewire/livewire
npm install
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### Пакети
- **Laravel 12** — Уеб рамка
- **Livewire 4** — Интерактивност без JavaScript
- **Tailwind CSS 4** — Стиловене
- **Stripe** — Плащания (по-късно)
- **Intervention Image** — Обработка на снимки

---

## Структура

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

## Стъпка по Стъпка

### Седмица 1: Основа
- [ ] Laravel проект
- [ ] База данни + миграции
- [ ] Модели (Product, Order)
- [ ] Главна страница

### Седмица 2: Продукти
- [ ] Листинг на продукти
- [ ] Страница продукт
- [ ] Качване на снимки

### Седмица 3: Кошница + Поръчка
- [ ] Кошница (Livewire)
- [ ] Форма за поръчка
- [ ] Запазване в база

### Седмица 4: Админ + Пускане
- [ ] Преглед на поръчки
- [ ] Промяна на статус
- [ ] Имейл потвърждение
- [ ] **ПУСКАНЕ**

---

## Код Примери

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
        
        // Запазване на продуктите от кошницата...
        
        // Изпращане на имейл...
        
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

## Плащане

### Вариант 1: Банков Превод (Сега)
- Клиентът вижда данните за плащане
- Плаща по банка
- Ние виждаме потвърждение и пращаме

### Вариант 2: Stripe (По-късно)
- Картови плащания
- Автоматично потвърждение

---

## Дизайн

### Принципи
- Много бяло пространство
- Една хубава снимка на продукт
- Ясни бутони
- Бързо зареждане
- Оптимизирано за мобилни

### Цветове
- Основен: Дърво (кафяво/бежово)
- Акцент: Зелен (действие)
- Текст: Тъмно сиво

---

## Снимки

Снимките са най-важни. Всеки продукт има 1 хубава снимка.

- Фон: Бял
- Светлина: Естествена
- Ъгъл: Равно отгоре
- Резолюция: 1200x1200

---

## Админ Панел

Само списък на поръчки:

```
| # | Клиент   | Продукти     | Статус | Действие         |
|---|----------|--------------|--------|------------------|
| 1 | Иван П.  | 2 бр         | Нова   | [Приеми] [Откажи]|
| 2 | Мария Д. | 1 бр         | Пратена| [Преглед]        |
```

Статуси: **Нова** → **В Работа** → **Готова** → **Пратена**

---

## Пускане

1. **Домейн**: gravir.pro
2. **Хостинг**: Shared или Vercel/Forge
3. **SSL**: Безплатен (Let's Encrypt)

---

## След Пускане

- [ ] Google Analytics — Проследяване на трафик
- [ ] Google Search Console — Индексиране
- [ ] Facebook Pixel — Реклама

---

## Команди

```bash
# Инсталиране
composer install
npm install

# Стартиране
php artisan serve
npm run dev

# Пушване в продакшън
npm run build
php artisan migrate
php artisan db:seed
```

---

## Какво След Пускане

След първите 50 поръчки добавяме:

1. [ ] Потребителски профили (вход/регистрация)
2. [ ] Отзиви от клиенти
3. [ ] Проследяване на поръчка
4. [ ] Плащане с карта (Stripe)
5. [ ] Персонализация на продукти (качване на снимки)
6. [ ] Еcont интеграция за куриер
