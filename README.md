# guide-fast-food
mon projet fast-food laravel

Parfait — je te donne **un guide clair, pas-à-pas et commenté** pour construire l’application **FastBite** en **Laravel**, expliqué comme à quelqu’un qui découvre l’informatique. Je vais **expliquer chaque action**, ce que ça fait et **montrer le code essentiel** (migrations, modèles, contrôleurs, routes, vues simples). À la fin tu sauras exactement quoi taper et pourquoi.

---

# Objectif simple

Construire une petite application web pour un fast-food où :

* un **client** peut voir le menu, ajouter au panier, valider une commande ;
* une **équipe** (cuisine / manager) peut voir les commandes et changer leur statut.

On commence **simple** (panier en session, paiement simulé), puis on pourra ajouter notifications réelles et paiements.

---

# Avant de commencer — vocabulaire très simple

* **PHP** : le langage serveur (comme « la cuisine » qui prépare les pages).
* **Composer** : gestionnaire de paquets PHP (comme un magasin pour installer du code).
* **Laravel** : le framework (un cadre qui facilite le développement).
* **MySQL / PostgreSQL** : la base de données (carnet où l’on écrit les commandes, produits).
* **Artisan** : l’outil en ligne de commande de Laravel (il crée fichiers, base, etc).
* **Migration** : un script qui crée des tables en base (comme un plan pour construire une table).
* **Model (Eloquent)** : représentation d’une table en PHP (une “classe” produit qui parle à la base).
* **Controller** : endroit où on met la logique (ce qu’il faut faire quand on clique sur “ajouter au panier”).
* **View (Blade)** : le code HTML visible par l’utilisateur (page produit, panier...).
* **Session** : un petit espace temporaire côté serveur pour stocker le panier d’un client.

---

# Pré-requis concrets (installation minimale)

Si ton ordinateur est neuf, installe :

1. PHP ≥ 8.1
2. Composer ([https://getcomposer.org](https://getcomposer.org))
3. Node.js + npm (pour le front build)
4. MySQL ou PostgreSQL
5. Git (optionnel mais recommandé)
6. Un éditeur : VS Code (recommandé)

Si tu veux, je peux te donner les commandes d’installation pour ton OS (Linux/Windows/Mac).

---

# Vue d’ensemble des étapes (ordre à suivre)

1. Créer le projet Laravel.
2. Configurer la base de données (.env).
3. Installer l’authentification (Breeze).
4. Créer les migrations (tables : categories, products, orders, order\_items, etc.).
5. Écrire les modèles (relations entre tables).
6. Créer contrôleurs (catalogue, panier, commande, admin).
7. Créer routes et vues (pages visibles).
8. Implémenter le panier (session).
9. Implémenter checkout (création d’une commande en base).
10. Tester (passer des commandes).
11. Ajouter admin, notifications, paiement (ensuite).
12. Déployer.

Je vais détailler **chaque étape** avec les commandes et le code.

---

# 1) Créer le projet Laravel — pas à pas

Ouvre un terminal et tape :

```bash
# Créer le projet
composer create-project laravel/laravel fastbite-laravel

cd fastbite-laravel
```

**Explication** : Composer télécharge Laravel et crée un dossier `fastbite-laravel` avec la structure du projet.

Démarre le serveur local (pour tester) :

```bash
php artisan serve
```

Ouvre `http://127.0.0.1:8000` dans le navigateur — tu dois voir la page Laravel.

---

# 2) Configurer la base de données

Ouvre le fichier `.env` (à la racine) et configure :

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=fastbite_db
DB_USERNAME=root
DB_PASSWORD=ton_mdp
```

Puis crée la base `fastbite_db` dans MySQL (via phpMyAdmin ou console MySQL).

**Explication** : `.env` contient les secrets et paramètres. On dit à Laravel où se trouve la base de données.

---

# 3) Installer l’authentification (utilisateurs)

On veut que les clients puissent se connecter / enregistrer. On installe **Breeze** (simple et clair) :

```bash
composer require laravel/breeze --dev
php artisan breeze:install
npm install
npm run dev
php artisan migrate
```

* `php artisan migrate` va appliquer les migrations par défaut (création table `users`, `password_resets`, etc).
* Après ça, la page d’accueil aura les liens `Login` / `Register`.

**Explication** : Breeze met en place un système d'inscription / login prêt à l'emploi (utile pour démarrer vite).

---

# 4) Création des migrations (tables principales)

On crée les tables de l’application. Dans le terminal :

```bash
php artisan make:model Category -m
php artisan make:model Product -m
php artisan make:model Order -m
php artisan make:model OrderItem -m
php artisan make:migration add_role_to_users_table --table=users
```

* Les fichiers `-m` sont des **migrations**. Ouvre-les dans `database/migrations/` et remplace leur contenu par ceci (exemples simplifiés).

**Migration `categories` :**

```php
// database/migrations/xxxx_create_categories_table.php
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug')->unique();
    $table->timestamps();
});
```

**Migration `products` :**

```php
Schema::create('products', function (Blueprint $table) {
    $table->id();
    $table->foreignId('category_id')->constrained()->cascadeOnDelete();
    $table->string('name');
    $table->text('description')->nullable();
    $table->decimal('price', 8, 2);
    $table->string('image')->nullable();
    $table->boolean('is_available')->default(true);
    $table->timestamps();
});
```

**Migration `orders` :**

```php
Schema::create('orders', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->decimal('total', 10, 2)->default(0);
    $table->string('status')->default('pending'); // pending, preparing, ready, delivered
    $table->text('address')->nullable();
    $table->string('payment_status')->default('unpaid');
    $table->timestamps();
});
```

**Migration `order_items` :**

```php
Schema::create('order_items', function (Blueprint $table) {
    $table->id();
    $table->foreignId('order_id')->constrained()->cascadeOnDelete();
    $table->foreignId('product_id')->constrained()->cascadeOnDelete();
    $table->integer('qty')->default(1);
    $table->decimal('price', 8, 2);
    $table->timestamps();
});
```

**Migration pour le rôle utilisateur :**

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('role')->default('client'); // client, admin, kitchen
});
```

Puis lance toutes les migrations :

```bash
php artisan migrate
```

**Explication** : Les migrations créent les tables en base. `foreignId(...)->constrained()` crée la clé étrangère. `migrate` exécute ces plans.

---

# 5) Les modèles (Eloquent) et leurs relations

Ouvre `app/Models/` et édite :

**Product.php**

```php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Model;

class Product extends Model {
    protected $fillable = ['category_id','name','description','price','image','is_available'];

    public function category() {
        return $this->belongsTo(Category::class);
    }
}
```

**Order.php**

```php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Model;

class Order extends Model {
    protected $fillable = ['user_id','total','status','address','payment_status'];

    public function items() {
        return $this->hasMany(OrderItem::class);
    }

    public function user() {
        return $this->belongsTo(User::class);
    }
}
```

**OrderItem.php**

```php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Model;

class OrderItem extends Model {
    protected $fillable = ['order_id','product_id','qty','price'];

    public function product() {
        return $this->belongsTo(Product::class);
    }
}
```

**Explication** : Les modèles permettent de manipuler les données facilement. `fillable` autorise l’insertion de ces champs en masse.

---

# 6) Routes et contrôleurs : comment réagir aux clics

Crée les contrôleurs (dans le terminal) :

```bash
php artisan make:controller ProductController --resource
php artisan make:controller CartController
php artisan make:controller OrderController
php artisan make:controller Admin/OrderAdminController
```

**Routes (fichier `routes/web.php`) — exemple minimal :**

```php
use App\Http\Controllers\ProductController;
use App\Http\Controllers\CartController;
use App\Http\Controllers\OrderController;

Route::get('/', [ProductController::class, 'index'])->name('home');

Route::resource('products', ProductController::class)->only(['index','show']);

Route::post('cart/add', [CartController::class,'add'])->name('cart.add');
Route::get('cart', [CartController::class,'index'])->name('cart.index');
Route::post('cart/remove', [CartController::class,'remove'])->name('cart.remove');

Route::middleware('auth')->group(function(){
    Route::post('order/checkout', [OrderController::class,'checkout'])->name('order.checkout');
    Route::get('orders', [OrderController::class,'index'])->name('orders.index'); // commandes user
});
```

**Explication** : Les routes connectent une URL à un contrôleur. Exemple : POST `cart/add` exécutera la fonction `add` du `CartController`.

---

# 7) Implémenter le panier (approche simple : session)

Le panier sera stocké en **session** (pratique pour débuter).

**Structure simple du panier (session)** :

```php
// session('cart') = [
/*
  product_id => [
    'id' => 12,
    'name' => 'Burger X',
    'price' => 500.00,
    'qty' => 2
  ],
  ...
*/
];
```

**CartController (extrait)** :

```php
<?php
namespace App\Http\Controllers;
use Illuminate\Http\Request;
use App\Models\Product;

class CartController extends Controller {
    public function index() {
        $cart = session('cart', []);
        $total = collect($cart)->reduce(function($carry, $item){
            return $carry + ($item['price'] * $item['qty']);
        }, 0);
        return view('cart.index', compact('cart','total'));
    }

    public function add(Request $request) {
        $product = Product::find($request->input('product_id'));
        if(!$product) return back()->with('error','Produit introuvable');

        $cart = session('cart', []);
        $id = $product->id;
        if(isset($cart[$id])) {
            $cart[$id]['qty'] += intval($request->input('qty',1));
        } else {
            $cart[$id] = [
                'id' => $product->id,
                'name' => $product->name,
                'price' => $product->price,
                'qty' => intval($request->input('qty',1))
            ];
        }
        session(['cart' => $cart]);
        return redirect()->route('cart.index')->with('success','Produit ajouté au panier');
    }

    public function remove(Request $request) {
        $id = $request->input('product_id');
        $cart = session('cart', []);
        if(isset($cart[$id])) {
            unset($cart[$id]);
            session(['cart' => $cart]);
        }
        return redirect()->route('cart.index');
    }
}
```

**Explication** : On utilise la session pour garder le panier même si l’utilisateur change de page. `session()` lit/écrit la session.

---

# 8) Checkout — création d’une commande en base (transaction)

Important : utiliser une **transaction** pour éviter d’avoir une commande partiellement créée.

**OrderController (extrait)** :

```php
<?php
namespace App\Http\Controllers;
use Illuminate\Http\Request;
use App\Models\Order;
use App\Models\OrderItem;
use Illuminate\Support\Facades\DB;

class OrderController extends Controller {
    public function checkout(Request $request) {
        $user = $request->user();
        $cart = session('cart', []);
        if(empty($cart)) return back()->with('error','Panier vide');

        DB::beginTransaction();
        try {
            $order = Order::create([
                'user_id' => $user->id,
                'address' => $request->input('address',''),
                'status' => 'pending',
                'payment_status' => 'unpaid'
            ]);
            $total = 0;
            foreach($cart as $item) {
                $price = $item['price'];
                $qty = $item['qty'];
                $order->items()->create([
                    'product_id' => $item['id'],
                    'qty' => $qty,
                    'price' => $price
                ]);
                $total += $price * $qty;
            }
            $order->update(['total' => $total, 'payment_status' => 'paid']); // ici on simule paiement
            DB::commit();
            session()->forget('cart');
            return redirect()->route('home')->with('success','Commande passée avec succès');
        } catch (\Throwable $e) {
            DB::rollBack();
            // log the error: \Log::error($e);
            return back()->with('error','Erreur lors du passsage de la commande');
        }
    }
}
```

**Explication** : `DB::beginTransaction()` ouvre une transaction : si une erreur survient, on annule tout (`rollBack`) pour ne pas casser l’état. Ici on simule que le paiement est `paid`.

---

# 9) Vues simples (Blade) — exemple rapide

**resources/views/products/index.blade.php**

```blade
@extends('layouts.app')

@section('content')
<h1>Menu</h1>
<div class="products">
  @foreach($products as $product)
    <div class="product">
      <h3>{{ $product->name }}</h3>
      <p>{{ $product->description }}</p>
      <p>{{ number_format($product->price,2) }} FCFA</p>
      <form action="{{ route('cart.add') }}" method="POST">
        @csrf
        <input type="hidden" name="product_id" value="{{ $product->id }}">
        <input type="number" name="qty" value="1" min="1">
        <button type="submit">Ajouter</button>
      </form>
    </div>
  @endforeach
</div>
{{ $products->links() }}
@endsection
```

**resources/views/cart/index.blade.php**

```blade
@extends('layouts.app')

@section('content')
<h1>Panier</h1>
@if(empty($cart))
  <p>Votre panier est vide</p>
@else
  <table>
    <tr><th>Produit</th><th>Qté</th><th>Prix</th><th></th></tr>
    @foreach($cart as $item)
      <tr>
        <td>{{ $item['name'] }}</td>
        <td>{{ $item['qty'] }}</td>
        <td>{{ number_format($item['price'] * $item['qty'],2) }}</td>
        <td>
          <form action="{{ route('cart.remove') }}" method="POST">
            @csrf
            <input type="hidden" name="product_id" value="{{ $item['id'] }}">
            <button type="submit">Retirer</button>
          </form>
        </td>
      </tr>
    @endforeach
  </table>
  <p>Total: {{ number_format($total,2) }} FCFA</p>
  @auth
    <form action="{{ route('order.checkout') }}" method="POST">
      @csrf
      <input type="text" name="address" placeholder="Adresse de livraison">
      <button type="submit">Commander</button>
    </form>
  @else
    <p><a href="{{ route('login') }}">Connecte-toi</a> pour valider la commande</p>
  @endauth
@endif
@endsection
```

**Explication** : Blade est le moteur de template. Ici on affiche produits et panier. Les formulaires envoient les actions aux routes définies.

---

# 10) Admin simple pour voir les commandes

Crée une route et un contrôleur Admin (protégez-les avec middleware `auth` + vérification `role == admin`).

**Exemple middleware simple dans controller :**

```php
public function __construct(){
    $this->middleware(function ($request, $next) {
        if(auth()->user()->role !== 'admin') {
            abort(403);
        }
        return $next($request);
    });
}
```

Dans `Admin/OrderAdminController`, crée une méthode `index()` pour lister les commandes et `updateStatus()` pour changer le statut.

---

# 11) Bonnes pratiques simples et erreurs courantes

* **Ne laisse jamais ton `.env` public** — il contient les clés.
* **APP\_DEBUG=false** en production (sinon les erreurs dévoilent des secrets).
* **Valide toujours** les données envoyées par l’utilisateur (`$request->validate([...])`).
* **Transactions** pour le checkout (évite commandes incomplètes).
* **Stock** : si tu veux un gestion de stock vraie, vérifie la disponibilité avant de valider l’ordre.
* **Tester** : crée des produits de test et passe des commandes pour vérifier chaque étape.
* **Logs** : regarde `storage/logs/laravel.log` si quelque chose casse.
* **Images** : `php artisan storage:link` pour lier le dossier public et stocker les images.

---

# 12) Ajouter plus tard (ordre recommandé)

1. Seeders & Factories (pour remplir la base en un clic).
2. Pagination & recherche pour le menu.
3. Notifications temps réel (Pusher + Laravel Echo) **ou** simple rafraîchissement toutes les 10s si débutant.
4. Intégration Stripe (mode test) : pour accepter les paiements en ligne.
5. Tests automatisés (PHPUnit).
6. Dockerisation (facile à déployer ensuite).
7. Déploiement (DigitalOcean, Render, etc).

---

# 13) Checklist pratique (à cocher)

* [ ] Installer PHP, Composer, Node, MySQL
* [ ] `composer create-project` (créer projet)
* [ ] `.env` configuré et `php artisan migrate` OK
* [ ] Auth avec Breeze OK (login/register)
* [ ] Migrations produits / orders / order\_items OK
* [ ] Pages produit / page panier OK
* [ ] Ajout au panier & suppression OK
* [ ] Checkout crée des ordres en base (transaction) OK
* [ ] Admin peut voir commandes et changer statut OK
* [ ] Tester plusieurs cas : produit supprimé, stock insuffisant, session vide

---

# 14) Conseils d’apprentissage — comment répéter pour “avoir la main”

* Implémente une **fonction à la fois** : d’abord afficher produits, ensuite le panier, puis le checkout.
* Après chaque petite fonctionnalité, **teste** (ouvre le navigateur, ajoute au panier, commande).
* Lis les erreurs dans `storage/logs` et Google l’erreur (StackOverflow souvent a la solution).
* Refais l’application **2 fois** : la première pour comprendre, la seconde pour optimiser et corriger.
* Ensuite, refais le même projet **en Django** — la répétition t’apportera la maîtrise.

---

# 15) Si tu veux le squelette complet (fichiers prêts)

Je peux **te fournir maintenant** :

* Les migrations complètes,
* Les modèles commentés,
* Les contrôleurs (ProductController, CartController, OrderController),
* Les vues Blade simples,
* Un Seeder pour remplir rapidement des produits.

Dis-moi juste : **“Oui, envoie le squelette Laravel”** et je te fournis les fichiers prêts à copier-coller (ou je peux les afficher directement ici).
(Je peux aussi les mettre dans un seul ZIP si tu veux ensuite.)

---

Si tu veux, je commence **tout de suite** et je te donne le **squelette complet** (migrations + models + controllers + vues) avec commentaires ligne par ligne. Veux-tu que je te l’envoie maintenant ?



Voici le **squelette complet** du projet **FastBite (Laravel)**, prêt à copier-coller dans ton dossier Laravel.
J’ai organisé les fichiers en **sections**, avec **commentaires** pour t’expliquer chaque partie.
👉 **But** : Tu colles ces fichiers dans ton projet (après avoir suivi l’installation du guide précédent) et tu exécutes `php artisan migrate` puis `php artisan serve` pour voir l’application.

---

## 📂 Structure du projet (à mettre dans `fastbite-laravel/`)

```
app/
 └─ Models/
      Category.php
      Product.php
      Order.php
      OrderItem.php
 └─ Http/
      Controllers/
          ProductController.php
          CartController.php
          OrderController.php
database/
 └─ migrations/
      2025_01_01_000001_create_categories_table.php
      2025_01_01_000002_create_products_table.php
      2025_01_01_000003_create_orders_table.php
      2025_01_01_000004_create_order_items_table.php
      2025_01_01_000005_add_role_to_users_table.php
resources/
 └─ views/
      layouts/app.blade.php
      products/index.blade.php
      cart/index.blade.php
routes/
 └─ web.php
```

---

### 1️⃣ Migrations – (dans `database/migrations/`)

📄 **create\_categories\_table.php**

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void {
        Schema::create('categories', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('slug')->unique();
            $table->timestamps();
        });
    }
    public function down(): void {
        Schema::dropIfExists('categories');
    }
};
```

📄 **create\_products\_table.php**

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void {
        Schema::create('products', function (Blueprint $table) {
            $table->id();
            $table->foreignId('category_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->text('description')->nullable();
            $table->decimal('price', 8, 2);
            $table->string('image')->nullable();
            $table->boolean('is_available')->default(true);
            $table->timestamps();
        });
    }
    public function down(): void {
        Schema::dropIfExists('products');
    }
};
```

📄 **create\_orders\_table.php**

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->decimal('total', 10, 2)->default(0);
            $table->string('status')->default('pending'); // pending, preparing, ready, delivered
            $table->text('address')->nullable();
            $table->string('payment_status')->default('unpaid');
            $table->timestamps();
        });
    }
    public function down(): void {
        Schema::dropIfExists('orders');
    }
};
```

📄 **create\_order\_items\_table.php**

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void {
        Schema::create('order_items', function (Blueprint $table) {
            $table->id();
            $table->foreignId('order_id')->constrained()->cascadeOnDelete();
            $table->foreignId('product_id')->constrained()->cascadeOnDelete();
            $table->integer('qty')->default(1);
            $table->decimal('price', 8, 2);
            $table->timestamps();
        });
    }
    public function down(): void {
        Schema::dropIfExists('order_items');
    }
};
```

📄 **add\_role\_to\_users\_table.php**

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void {
        Schema::table('users', function (Blueprint $table) {
            $table->string('role')->default('client'); // client, admin
        });
    }
    public function down(): void {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('role');
        });
    }
};
```

---

### 2️⃣ Modèles – (dans `app/Models/`)

📄 **Category.php**

```php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Model;

class Category extends Model {
    protected $fillable = ['name','slug'];

    public function products() {
        return $this->hasMany(Product::class);
    }
}
```

📄 **Product.php**

```php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Model;

class Product extends Model {
    protected $fillable = ['category_id','name','description','price','image','is_available'];

    public function category() {
        return $this->belongsTo(Category::class);
    }
}
```

📄 **Order.php**

```php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Model;

class Order extends Model {
    protected $fillable = ['user_id','total','status','address','payment_status'];

    public function items() {
        return $this->hasMany(OrderItem::class);
    }
    public function user() {
        return $this->belongsTo(User::class);
    }
}
```

📄 **OrderItem.php**

```php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Model;

class OrderItem extends Model {
    protected $fillable = ['order_id','product_id','qty','price'];

    public function product() {
        return $this->belongsTo(Product::class);
    }
}
```

---

### 3️⃣ Contrôleurs – (dans `app/Http/Controllers/`)

📄 **ProductController.php**

```php
<?php
namespace App\Http\Controllers;

use App\Models\Product;

class ProductController extends Controller {
    // Affiche le menu des produits
    public function index() {
        $products = Product::where('is_available',true)->paginate(8);
        return view('products.index', compact('products'));
    }
}
```

📄 **CartController.php**

```php
<?php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\Product;

class CartController extends Controller {

    public function index() {
        $cart = session('cart', []);
        $total = collect($cart)->sum(fn($i) => $i['price'] * $i['qty']);
        return view('cart.index', compact('cart','total'));
    }

    public function add(Request $request) {
        $product = Product::find($request->input('product_id'));
        if(!$product) return back()->with('error','Produit introuvable');

        $cart = session('cart', []);
        $id = $product->id;
        $cart[$id]['id']    = $product->id;
        $cart[$id]['name']  = $product->name;
        $cart[$id]['price'] = $product->price;
        $cart[$id]['qty']   = ($cart[$id]['qty'] ?? 0) + (int)$request->input('qty',1);

        session(['cart' => $cart]);
        return redirect()->route('cart.index')->with('success','Produit ajouté au panier');
    }

    public function remove(Request $request) {
        $cart = session('cart', []);
        unset($cart[$request->input('product_id')]);
        session(['cart' => $cart]);
        return redirect()->route('cart.index');
    }
}
```

📄 **OrderController.php**

```php
<?php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use App\Models\Order;

class OrderController extends Controller {
    public function index() {
        $orders = auth()->user()->orders()->latest()->get();
        return view('orders.index', compact('orders'));
    }

    public function checkout(Request $request) {
        $user = $request->user();
        $cart = session('cart', []);
        if(empty($cart)) return back()->with('error','Panier vide');

        DB::beginTransaction();
        try {
            $order = Order::create([
                'user_id' => $user->id,
                'address' => $request->input('address',''),
                'status'  => 'pending',
                'payment_status' => 'paid' // simulé
            ]);

            $total = 0;
            foreach($cart as $item){
                $order->items()->create([
                    'product_id' => $item['id'],
                    'qty'        => $item['qty'],
                    'price'      => $item['price'],
                ]);
                $total += $item['price'] * $item['qty'];
            }

            $order->update(['total' => $total]);
            DB::commit();
            session()->forget('cart');
            return redirect()->route('home')->with('success','Commande passée');
        } catch (\Throwable $e) {
            DB::rollBack();
            return back()->with('error','Erreur : '.$e->getMessage());
        }
    }
}
```

---

### 4️⃣ Routes – (dans `routes/web.php`)

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\ProductController;
use App\Http\Controllers\CartController;
use App\Http\Controllers\OrderController;

Route::get('/', [ProductController::class, 'index'])->name('home');

Route::post('cart/add', [CartController::class,'add'])->name('cart.add');
Route::get('cart', [CartController::class,'index'])->name('cart.index');
Route::post('cart/remove', [CartController::class,'remove'])->name('cart.remove');

Route::middleware('auth')->group(function(){
    Route::get('orders', [OrderController::class,'index'])->name('orders.index');
    Route::post('order/checkout', [OrderController::class,'checkout'])->name('order.checkout');
});
```

---

### 5️⃣ Vues Blade – (dans `resources/views/`)

📄 **layouts/app.blade.php**
*(un layout simple — tu peux remplacer par celui généré par Breeze)*

```blade
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>FastBite</title>
    @vite(['resources/css/app.css','resources/js/app.js'])
</head>
<body class="p-4">
<nav>
    <a href="{{ route('home') }}">Menu</a> |
    <a href="{{ route('cart.index') }}">Panier ({{ count(session('cart', [])) }})</a>
    @auth
        | <a href="{{ route('orders.index') }}">Mes commandes</a>
        | <form action="/logout" method="POST" style="display:inline">@csrf<button>Déconnexion</button></form>
    @else
        | <a href="/login">Connexion</a> | <a href="/register">Inscription</a>
    @endauth
</nav>
<hr>
@if(session('success'))<p style="color:green">{{ session('success') }}</p>@endif
@if(session('error'))<p style="color:red">{{ session('error') }}</p>@endif
@yield('content')
</body>
</html>
```

📄 **products/index.blade.php**

```blade
@extends('layouts.app')

@section('content')
<h1>Menu FastBite</h1>
<div style="display:flex;flex-wrap:wrap;gap:20px">
@foreach($products as $product)
  <div style="border:1px solid #ccc;padding:10px;width:200px">
    <h3>{{ $product->name }}</h3>
    <p>{{ $product->description }}</p>
    <p><strong>{{ number_format($product->price,2) }} FCFA</strong></p>
    <form action="{{ route('cart.add') }}" method="POST">
      @csrf
      <input type="hidden" name="product_id" value="{{ $product->id }}">
      <input type="number" name="qty" value="1" min="1" style="width:60px">
      <button type="submit">Ajouter</button>
    </form>
  </div>
@endforeach
</div>
{{ $products->links() }}
@endsection
```

📄 **cart/index.blade.php**

```blade
@extends('layouts.app')

@section('content')
<h1>Votre panier</h1>
@if(empty($cart))
  <p>Panier vide</p>
@else
<table border="1" cellpadding="5">
<tr><th>Produit</th><th>Qté</th><th>Prix</th><th>Action</th></tr>
@foreach($cart as $item)
<tr>
  <td>{{ $item['name'] }}</td>
  <td>{{ $item['qty'] }}</td>
  <td>{{ number_format($item['price'] * $item['qty'],2) }}</td>
  <td>
    <form action="{{ route('cart.remove') }}" method="POST">
      @csrf
      <input type="hidden" name="product_id" value="{{ $item['id'] }}">
      <button>Retirer</button>
    </form>
  </td>
</tr>
@endforeach
</table>
<p><strong>Total : {{ number_format($total,2) }} FCFA</strong></p>

@auth
<form action="{{ route('order.checkout') }}" method="POST">
  @csrf
  <input type="text" name="address" placeholder="Adresse de livraison" required>
  <button>Commander</button>
</form>
@else
<p><a href="/login">Connecte-toi</a> pour commander</p>
@endauth
@endif
@endsection
```

---

## 🚀 Mise en route

1. Copie ces fichiers dans ton projet Laravel.
2. Lance les commandes :

```bash
composer install
npm install && npm run dev
php artisan migrate
php artisan serve
```

3. Crée un compte utilisateur via `/register`.
4. Dans phpMyAdmin, ajoute manuellement quelques **categories** et **products** pour tester.

---

## ✅ Résultat

* Page d’accueil `/` : affiche le **menu** (produits).
* Page `/cart` : affiche le **panier**.
* Login/Register : déjà prêt (via Breeze).
* Checkout : crée une commande dans la base.
* Les admins (role `admin`) pourront plus tard avoir une page spéciale pour gérer les commandes.

---

💡 **Prochaines étapes**

* Ajouter un `Seeder` pour créer des produits automatiquement.
* Créer une interface Admin.
* Ajouter le téléchargement d’images pour les produits.
* Passer en production (hébergement).

---

Ce squelette te donne **100% des fondations**.
➡️ **Répète l’installation** plusieurs fois (supprimer/recréer le projet) pour bien comprendre les étapes.

