# Допускаем что проект реализовано на  Laravel

## 1. Модификация каталога товаров (Избранное)

### 1.1 Создаем таблицу для хранения избранных товаров:

```php
Schema::create('favorites', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->foreignId('product_id')->constrained()->onDelete('cascade');
    $table->timestamps();

    $table->unique(['user_id', 'product_id']);
});
```
### 1.2 Создаем модель для работы с избранными товарами:

```php
class Favorite extends Model
{
    protected $fillable = ['user_id', 'product_id'];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function product(): BelongsTo
    {
        return $this->belongsTo(Product::class);
    }
}
```
### 1.3 Создаем роуты для работы с избранными товарами:

```php
Route::middleware('auth:sanctum')->group(function () {
    Route::post('/favorites/{product}', [FavoriteController::class, 'store']);
    Route::delete('/favorites/{product}', [FavoriteController::class, 'destroy']);
});
```

### 1.4 Создаем контроллер для работы с избранными товарами:

```php
class FavoriteController extends Controller
{
    public function index(Request $request)
    {
        $products = Product::whereHas('favorites', fn ($q) => $q->where('user_id', auth()->id()))
            ->applyFilters($request)
            ->paginate(15);
    
        return ProductResource::collection($products);
    }

    public function store(Product $product)
    {
        Favorite::firstOrCreate([
            'user_id' => auth()->id(),
            'product_id' => $product->id,
        ]);

        return response()->json([
            'message' => 'Product added to favorites',
            'product' => $product,
        ], 201);
    }

    public function destroy(Product $product)
    {
        Favorite::where('user_id', auth()->id())
            ->where('product_id', $product->id)
            ->delete();

        return response()->json([
            'message' => 'Product removed from favorites',
        ], 200);
    }
}
```

### 1.5 Добавляем в Product модель связь с избранными товарами:

```php
public function favorites(): HasMany
{
    return $this->hasMany(Favorite::class);
}
```

### 1.6 Добавляем в ProductResource отметку избранного:

```php
'is_favorite' => $this->relationLoaded('favorites')
    ? $this->favorites->contains('user_id', auth()->id())
    : false,
```
## 2. Модификация каталога товаров (Изображения)

### 2.1 Модификация таблицы изображений товаров добавляем поле `source` для указания источника изображения (по умолчанию делаем, 'local'):
```php
Schema::table('product_images', function (Blueprint $table) {
    $table->string('source')->default('local');
});
```

### 2.2 Изменяем модель ProductImage для работы с источником изображений:

```php
class ProductImage extends Model
{
    protected $fillable = ['product_id', 'path', 'source']; // source = 'local'|'s3'

    public function product(): BelongsTo
    {
        return $this->belongsTo(Product::class);
    }

    public function getUrlAttribute(): string
    {
        return match ($this->source) {
            'local' => Storage::url($this->path),
            's3' => Storage::disk('s3')->url($this->path),
            default => $this->path,
        };
    }
}
```
### 2.3 Добавдяем связь в Product модель для изображений:

```php
public function images(): HasMany
{
    return $this->hasMany(ProductImage::class);
}
```
### 2.4 Обновляем ProductResource для отображения изображений с учетом источника:

```php
[
    //...
    'image_url' => $this->images->first()?->url, // для обратной совместимости
    'images' => $this->images->map(fn($image) => [
        'url' => $image->url,
    ]),
    //...
]
```

### Пример API-ответа (GET /products)
```json
[
  {
    "id": 1,
    "name": "Example product 1",
    "description": "Example product 1 description",
    "category": "category-1",
    "image_url": "https://cdn.market.com/images/products/product_1.png",
    "images": [
      { "url": "https://cdn.market.com/images/products/product_1.png" },
      { "url": "https://s3.amazonaws.com/bucket/image1-alt.jpg" }
    ],
    "is_favorite": true
  },
  {
    "id": 4,
    "name": "Example product 4",
    "description": "Example product 4 description",
    "category": "category-1",
    "image_url": "https://cdn.market.com/images/products/product_4.png",
    "images": [
      { "url": "https://cdn.market.com/images/products/product_4.png" }
    ],
    "is_favorite": false
  }
]

```
---------
## Релизация: Интерфейс для работы с изображениями 
```php
namespace Market;

interface ImageStorageInterface
{
    public function getUrl(string $fileName): ?string;

    public function fileExists(string $fileName): bool;

    public function deleteFile(string $fileName): void;

    public function saveFile(string $fileName): void;
}

namespace Market;

class LocalStorageAdapter implements ImageStorageInterface
{
    public function __construct(private FileStorageRepository $repo)
    {
    }

    public function getUrl(string $fileName): ?string
    {
        return $this->repo->getUrl($fileName);
    }

    public function fileExists(string $fileName): bool
    {
        return $this->repo->fileExists($fileName);
    }

    public function deleteFile(string $fileName): void
    {
        $this->repo->deleteFile($fileName);
    }

    public function saveFile(string $fileName): void
    {
        $this->repo->saveFile($fileName);
    }
}

namespace Market;

use AwsS3\Client\AwsStorageInterface;
use AwsS3\AwsUrlInterface;
use Exception;

class AwsStorageAdapter implements ImageStorageInterface
{
    public function __construct(private AwsStorageInterface $aws)
    {
    }

    public function getUrl(string $fileName): ?string
    {
        try {
            /** @var AwsUrlInterface $url */
            $url = $this->aws->getUrl($fileName);
            return (string) $url;
        } catch (Exception $e) {
            return null;
        }
    }

    public function fileExists(string $fileName): bool
    {
        try {
            $this->aws->getUrl($fileName);
            return true;
        } catch (Exception $e) {
            return false;
        }
    }

    public function deleteFile(string $fileName): void
    {
    }

    public function saveFile(string $fileName): void
    {
    }
}

namespace Market;

class Product
{
    private string $imageFileName;

    public function __construct(
        private ImageStorageInterface $storage
    ) {}

    public function getImageUrl(): ?string
    {
        if (!$this->storage->fileExists($this->imageFileName)) {
            return null;
        }

        return $this->storage->getUrl($this->imageFileName);
    }

    public function updateImage(): bool
    {
        try {
            if ($this->storage->fileExists($this->imageFileName)) {
                $this->storage->deleteFile($this->imageFileName);
            }

            $this->storage->saveFile($this->imageFileName);
        } catch (\Exception $e) {
            return false;
        }

        return true;
    }
}


```


---------
## Структуры корзины заказов
```php
readonly class CartItem
{
    public function __construct(
        private int    $id,
        private string $name,
        private float  $price,
        private int    $quantity = 1
    ) {}

    public function getId(): int       { return $this->id; }
    public function getName(): string  { return $this->name; }
    public function getPrice(): float  { return $this->price; }
    public function getQuantity(): int { return $this->quantity; }
    public function getTotal(): float  { return $this->price * $this->quantity; }
}



class OrderCart
{
    /** @var CartItem[] */
    protected array $items = [];

    public function addItem(CartItem $item): void
    {
        foreach ($this->items as $index => $existing) {
            if ($existing->getId() === $item->getId()) {
                $newQuantity = $existing->getQuantity() + $item->getQuantity();

                $this->items[$index] = new CartItem(
                    $existing->getId(),
                    $existing->getName(),
                    $existing->getPrice(),
                    $newQuantity
                );
                return;
            }
        }

        $this->items[] = $item;
    }

    public function deleteItem(CartItem $item): void
    {
        $this->items = array_filter($this->items, fn($i) => $i->getId() !== $item->getId());
    }

    public function getItems(): array
    {
        return $this->items;
    }

    public function getItemsCount(): int
    {
        return array_sum(array_map(fn($i) => $i->getQuantity(), $this->items));
    }

    public function calculateTotalSum(): float
    {
        return array_sum(array_map(fn($i) => $i->getTotal(), $this->items));
    }

    public function printOrder(): void
    {
        echo "Order Summary:\n";
        foreach ($this->items as $item) {
            echo "- {$item->getName()} x{$item->getQuantity()} = {$item->getTotal()}\n";
        }
        echo "Total: {$this->calculateTotalSum()}\n";
    }

    public function showOrder(): void
    {
        $this->printOrder();
    }

    public function save(): void
    {
        $_SESSION['order_cart'] = serialize($this);
    }

    public function load(): void
    {
        if (isset($_SESSION['order_cart'])) {
            $saved = unserialize($_SESSION['order_cart']);
            $this->items = $saved->items ?? [];
        }
    }

    public function update(): void
    {
        $this->save();
    }

    public function delete(): void
    {
        unset($_SESSION['order_cart']);
        $this->items = [];
    }
}
```
---------
## Репозиторий билетов
```php
interface TicketDataSourceInterface
{
    public function load(int $ticketId): ?Ticket;
    public function save(Ticket $ticket): bool;
    public function update(Ticket $ticket): bool;
    public function delete(Ticket $ticket): bool;
}

class DatabaseTicketDataSource implements TicketDataSourceInterface
{
    public function load(int $ticketId): ?Ticket
    {
        return Ticket::find()->where(['id' => $ticketId])->one();
    }

    public function save(Ticket $ticket): bool
    {
        return $ticket->save();
    }

    public function update(Ticket $ticket): bool
    {
        return $ticket->update();
    }

    public function delete(Ticket $ticket): bool
    {
        return $ticket->delete();
    }
}

class ApiTicketDataSource implements TicketDataSourceInterface
{
    protected string $apiBaseUrl = 'https://api.example.com/tickets';

    public function load(int $ticketId): ?Ticket
    {
        $ch = curl_init("{$this->apiBaseUrl}/{$ticketId}");
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        $response = curl_exec($ch);
        curl_close($ch);
    
        $data = json_decode($response, true);
    
        return $data ? new Ticket($data) : null;
    }

    public function save(Ticket $ticket): bool
    {
        return true;
    }

    public function update(Ticket $ticket): bool
    {
        return true;
    }

    public function delete(Ticket $ticket): bool
    {
        return true;
    }
}

readonly class TicketRepository
{
    public function __construct(private TicketDataSourceInterface $dataSource)
    {
    }

    public function load(int $ticketId): ?Ticket
    {
        return $this->dataSource->load($ticketId);
    }

    public function save(Ticket $ticket): bool
    {
        return $this->dataSource->save($ticket);
    }

    public function update(Ticket $ticket): bool
    {
        return $this->dataSource->update($ticket);
    }

    public function delete(Ticket $ticket): bool
    {
        return $this->dataSource->delete($ticket);
    }
}
```
---------
## Composer: Обновление зависимости

Допустим, у нас есть библиотека vendor/package.

Клонируем библиотеку:
```bash
git clone git@github.com:vendor/package.git
cd package
```
Создаём ветку:
```bash
git checkout -b feature
```
Вносим изменения и коммитим их:
```bash
git add .
git commit -m "Fix bug in package feature"
```
В основном проекте подключаем локальную версию:
```json
"repositories": [
  {
    "type": "path",
    "url": "../<LOCAL_PATH_TO_OUR_PACKAGE>"
  }
]
```
Меняем версию библиотеки:

```bash
composer require vendor/package:dev-feature
```
Проверяем работоспособность если всё работает, тогда релизим новую версию библиотеки:
сначала меняем в `composer.json` версию
```json
{
  "version": "1.1.0"
}
```
Создаём git-тег:
```bash
git tag v1.1.0
git push origin main
git push origin v1.1.0
```
Обновление библиотеки в проекте
```bash
composer update vendor/package
```
---------
## SQL: Оценки студентов
```sql
SELECT
    CASE
        WHEN g.grade < 8 THEN 'low'
        ELSE s.name
        END AS name,
    g.grade,
    s.marks
FROM
    students s
        JOIN
    grade g ON s.marks BETWEEN g.min_mark AND g.max_mark
ORDER BY
    CASE
        WHEN g.grade >= 8 THEN s.marks
        END DESC,
    CASE
        WHEN g.grade >= 8 THEN s.name
        END ASC,
    CASE
        WHEN g.grade < 8 THEN g.grade
        END DESC,
    CASE
        WHEN g.grade < 8 THEN s.marks
        END ASC;
```
### Модификация DDL
```sql
CREATE TABLE grade (
  grade INT PRIMARY KEY,
  min_mark INT NOT NULL,
  max_mark INT NOT NULL
);

CREATE TABLE students (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  marks INT NOT NULL,
  INDEX idx_marks (marks)
);
```
--------
## Docker: Модификация конфигурации сервисов
В корне проекта создаем файл `docker-compose.override.yml` для переопределения конфигурации сервисов:

```yaml
version: '3'
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: app
      MYSQL_USER: app
      MYSQL_PASSWORD: secret
    volumes:
      - ./etc/infrastructure/mysql/my.cnf:/etc/mysql/my.cnf:ro
      - ./etc/database/base.sql:/docker-entrypoint-initdb.d/base.sql

  php:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: app-php
    volumes:
      - ./:/var/www
    depends_on:
      - db

  nginx:
    volumes:
      - ./:/var/www
      - ./etc/infrastructure/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - php

networks:
  default:
    name: app-network
```
