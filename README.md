# Laravel Cheat Sheet

A comprehensive reference for Laravel - a PHP web framework with elegant syntax for rapid application development.

---

## Table of Contents
- [Installation and Setup](#installation-and-setup)
- [Routing](#routing)
- [Controllers](#controllers)
- [Models and Eloquent ORM](#models-and-eloquent-orm)
- [Blade Templates](#blade-templates)
- [Forms and Validation](#forms-and-validation)
- [Authentication](#authentication)
- [Database and Migrations](#database-and-migrations)
- [Middleware](#middleware)
- [Artisan Commands](#artisan-commands)
- [Testing](#testing)
- [Deployment](#deployment)

---

## Installation and Setup

### Installation
```bash
# Install via Composer
composer create-project laravel/laravel my-project

# Or using Laravel installer
composer global require laravel/installer
laravel new my-project

# Start development server
php artisan serve

# Generate application key
php artisan key:generate
```

### Project Structure
```
app/
├── Console/
├── Exceptions/
├── Http/
│   ├── Controllers/
│   ├── Middleware/
│   └── Kernel.php
├── Models/
└── Providers/
config/
database/
├── factories/
├── migrations/
└── seeders/
resources/
├── views/
├── js/
└── css/
routes/
├── web.php
└── api.php
storage/
tests/
```

### Environment Configuration
```env
# .env
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:key
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=

CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
```

## Routing

### Basic Routing
```php
// routes/web.php
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\PostController;

// Basic routes
Route::get('/', function () {
    return view('welcome');
});

Route::get('/about', function () {
    return view('about');
});

// Route parameters
Route::get('/user/{id}', function ($id) {
    return 'User ID: ' . $id;
});

Route::get('/user/{name?}', function ($name = 'Guest') {
    return 'Hello, ' . $name;
});

// Route with constraints
Route::get('/user/{id}', function ($id) {
    return 'User ID: ' . $id;
})->where('id', '[0-9]+');

Route::get('/user/{name}', function ($name) {
    return 'User: ' . $name;
})->where('name', '[A-Za-z]+');

// Named routes
Route::get('/profile', function () {
    return view('profile');
})->name('profile');

// Route groups
Route::prefix('admin')->group(function () {
    Route::get('/users', function () {
        return 'Admin Users';
    });
    Route::get('/posts', function () {
        return 'Admin Posts';
    });
});

// Middleware group
Route::middleware(['auth'])->group(function () {
    Route::get('/dashboard', function () {
        return view('dashboard');
    });
});

// Controller routes
Route::get('/posts', [PostController::class, 'index']);
Route::post('/posts', [PostController::class, 'store']);
Route::resource('posts', PostController::class);

// API routes (routes/api.php)
Route::apiResource('posts', PostController::class);
```

## Controllers

### Basic Controllers
```php
// app/Http/Controllers/PostController.php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;
use Illuminate\View\View;
use Illuminate\Http\RedirectResponse;

class PostController extends Controller
{
    public function index(): View
    {
        $posts = Post::latest()->paginate(10);
        return view('posts.index', compact('posts'));
    }
    
    public function show(Post $post): View
    {
        return view('posts.show', compact('post'));
    }
    
    public function create(): View
    {
        return view('posts.create');
    }
    
    public function store(Request $request): RedirectResponse
    {
        $request->validate([
            'title' => 'required|max:255',
            'content' => 'required',
        ]);
        
        Post::create([
            'title' => $request->title,
            'content' => $request->content,
            'user_id' => auth()->id(),
        ]);
        
        return redirect()->route('posts.index')
            ->with('success', 'Post created successfully!');
    }
    
    public function edit(Post $post): View
    {
        $this->authorize('update', $post);
        return view('posts.edit', compact('post'));
    }
    
    public function update(Request $request, Post $post): RedirectResponse
    {
        $this->authorize('update', $post);
        
        $request->validate([
            'title' => 'required|max:255',
            'content' => 'required',
        ]);
        
        $post->update($request->only(['title', 'content']));
        
        return redirect()->route('posts.show', $post)
            ->with('success', 'Post updated successfully!');
    }
    
    public function destroy(Post $post): RedirectResponse
    {
        $this->authorize('delete', $post);
        $post->delete();
        
        return redirect()->route('posts.index')
            ->with('success', 'Post deleted successfully!');
    }
}

// API Controller
class ApiPostController extends Controller
{
    public function index()
    {
        return Post::with('user')->latest()->paginate(10);
    }
    
    public function store(Request $request)
    {
        $request->validate([
            'title' => 'required|max:255',
            'content' => 'required',
        ]);
        
        $post = Post::create([
            'title' => $request->title,
            'content' => $request->content,
            'user_id' => auth()->id(),
        ]);
        
        return response()->json($post, 201);
    }
}
```

## Models and Eloquent ORM

### Eloquent Models
```php
// app/Models/Post.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Post extends Model
{
    use HasFactory, SoftDeletes;
    
    protected $fillable = [
        'title',
        'slug',
        'content',
        'excerpt',
        'published_at',
        'user_id',
        'category_id',
    ];
    
    protected $casts = [
        'published_at' => 'datetime',
        'is_published' => 'boolean',
    ];
    
    protected $dates = ['deleted_at'];
    
    // Relationships
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
    
    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }
    
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }
    
    public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class);
    }
    
    // Accessors & Mutators
    public function getExcerptAttribute(): string
    {
        return $this->excerpt ?: substr(strip_tags($this->content), 0, 100) . '...';
    }
    
    public function setTitleAttribute(string $value): void
    {
        $this->attributes['title'] = $value;
        $this->attributes['slug'] = str()->slug($value);
    }
    
    // Scopes
    public function scopePublished($query)
    {
        return $query->whereNotNull('published_at')
                    ->where('published_at', '<=', now());
    }
    
    public function scopeByCategory($query, $category)
    {
        return $query->where('category_id', $category);
    }
    
    // Model Events
    protected static function booted()
    {
        static::creating(function ($post) {
            $post->user_id = auth()->id();
        });
    }
}

// Query Examples
$posts = Post::with(['user', 'category', 'tags'])
    ->published()
    ->latest()
    ->paginate(10);

$post = Post::where('slug', $slug)->firstOrFail();

$userPosts = User::find(1)->posts()->published()->get();

// Advanced queries
$posts = Post::whereHas('tags', function ($query) {
    $query->where('name', 'laravel');
})->get();

$posts = Post::withCount('comments')
    ->orderBy('comments_count', 'desc')
    ->get();
```

### Relationships
```php
// One-to-Many
class User extends Model
{
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }
}

class Post extends Model
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}

// Many-to-Many
class Post extends Model
{
    public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class)
                    ->withPivot('created_at')
                    ->withTimestamps();
    }
}

class Tag extends Model
{
    public function posts(): BelongsToMany
    {
        return $this->belongsToMany(Post::class);
    }
}

// Polymorphic Relations
class Comment extends Model
{
    public function commentable()
    {
        return $this->morphTo();
    }
}

class Post extends Model
{
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

## Blade Templates

### Blade Syntax
```php
<!-- resources/views/layouts/app.blade.php -->
<!DOCTYPE html>
<html>
<head>
    <title>@yield('title', 'Laravel App')</title>
    <meta name="csrf-token" content="{{ csrf_token() }}">
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    <nav class="navbar">
        <a href="{{ route('home') }}">Home</a>
        @auth
            <a href="{{ route('dashboard') }}">Dashboard</a>
            <form method="POST" action="{{ route('logout') }}">
                @csrf
                <button type="submit">Logout</button>
            </form>
        @else
            <a href="{{ route('login') }}">Login</a>
        @endauth
    </nav>
    
    <main>
        @if(session('success'))
            <div class="alert alert-success">
                {{ session('success') }}
            </div>
        @endif
        
        @yield('content')
    </main>
</body>
</html>

<!-- resources/views/posts/index.blade.php -->
@extends('layouts.app')

@section('title', 'Posts')

@section('content')
<div class="container">
    <h1>Posts</h1>
    
    @can('create', App\Models\Post::class)
        <a href="{{ route('posts.create') }}" class="btn btn-primary">Create Post</a>
    @endcan
    
    @forelse($posts as $post)
        <article class="post">
            <h2>
                <a href="{{ route('posts.show', $post) }}">{{ $post->title }}</a>
            </h2>
            <p>{{ $post->excerpt }}</p>
            <small>
                By {{ $post->user->name }} on {{ $post->created_at->format('M j, Y') }}
            </small>
            
            @if($post->tags->count() > 0)
                <div class="tags">
                    @foreach($post->tags as $tag)
                        <span class="tag">{{ $tag->name }}</span>
                    @endforeach
                </div>
            @endif
        </article>
    @empty
        <p>No posts found.</p>
    @endforelse
    
    {{ $posts->links() }}
</div>
@endsection

<!-- Blade Components -->
<!-- resources/views/components/alert.blade.php -->
@props(['type' => 'info', 'message'])

<div class="alert alert-{{ $type }}">
    {{ $message ?? $slot }}
</div>

<!-- Usage -->
<x-alert type="success" message="Post created successfully!" />
<x-alert type="error">
    Something went wrong!
</x-alert>
```

### Blade Directives
```php
<!-- Conditionals -->
@if($user->isAdmin())
    <p>Admin content</p>
@elseif($user->isModerator())
    <p>Moderator content</p>
@else
    <p>Regular user content</p>
@endif

@unless($user->isAdmin())
    <p>You are not an admin</p>
@endunless

@isset($post)
    <h1>{{ $post->title }}</h1>
@endisset

@empty($posts)
    <p>No posts available</p>
@endempty

<!-- Authentication -->
@auth
    <p>Welcome, {{ auth()->user()->name }}!</p>
@endauth

@guest
    <a href="{{ route('login') }}">Login</a>
@endguest

@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan

<!-- Loops -->
@foreach($posts as $post)
    <div>{{ $post->title }}</div>
@endforeach

@forelse($posts as $post)
    <div>{{ $post->title }}</div>
@empty
    <div>No posts</div>
@endforelse

@for($i = 0; $i < 10; $i++)
    <p>Item {{ $i }}</p>
@endfor

@while($condition)
    <!-- Content -->
@endwhile

<!-- Loop variables -->
@foreach($posts as $post)
    @if($loop->first)
        <p>This is the first post</p>
    @endif
    
    <div>Post {{ $loop->iteration }}: {{ $post->title }}</div>
    
    @if($loop->last)
        <p>This is the last post</p>
    @endif
@endforeach

<!-- Including views -->
@include('partials.header')
@include('partials.post', ['post' => $post])

<!-- Sections and yields -->
@section('sidebar')
    <p>Default sidebar content</p>
@endsection

@section('content')
    <p>Page content</p>
@endsection

<!-- In child template -->
@section('sidebar')
    @parent
    <p>Additional sidebar content</p>
@endsection
```

## Forms and Validation

### Form Handling
```php
// Controller
public function store(Request $request)
{
    $validated = $request->validate([
        'title' => 'required|string|max:255',
        'email' => 'required|email|unique:users,email',
        'password' => 'required|min:8|confirmed',
        'age' => 'required|integer|min:18|max:100',
        'avatar' => 'nullable|image|max:2048',
        'tags' => 'array',
        'tags.*' => 'string|max:50',
    ]);
    
    // Custom validation messages
    $request->validate([
        'title' => 'required|max:255',
    ], [
        'title.required' => 'The title field is mandatory.',
        'title.max' => 'The title cannot exceed 255 characters.',
    ]);
    
    // Custom validation rules
    $request->validate([
        'email' => ['required', 'email', new CustomEmailRule],
    ]);
}

// Form Request
// php artisan make:request StorePostRequest
class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }
    
    public function rules(): array
    {
        return [
            'title' => 'required|string|max:255',
            'content' => 'required|string',
            'category_id' => 'required|exists:categories,id',
            'tags' => 'array',
            'tags.*' => 'string|max:50',
        ];
    }
    
    public function messages(): array
    {
        return [
            'title.required' => 'Please enter a title for your post.',
            'content.required' => 'Post content is required.',
        ];
    }
    
    public function attributes(): array
    {
        return [
            'category_id' => 'category',
        ];
    }
}

// Usage in controller
public function store(StorePostRequest $request)
{
    $post = Post::create($request->validated());
    return redirect()->route('posts.show', $post);
}
```

### Form Templates
```php
<!-- resources/views/posts/create.blade.php -->
@extends('layouts.app')

@section('content')
<form method="POST" action="{{ route('posts.store') }}" enctype="multipart/form-data">
    @csrf
    
    <div class="form-group">
        <label for="title">Title</label>
        <input type="text" 
               id="title" 
               name="title" 
               value="{{ old('title') }}"
               class="form-control @error('title') is-invalid @enderror"
               required>
        @error('title')
            <div class="invalid-feedback">{{ $message }}</div>
        @enderror
    </div>
    
    <div class="form-group">
        <label for="content">Content</label>
        <textarea id="content" 
                  name="content" 
                  class="form-control @error('content') is-invalid @enderror"
                  rows="10">{{ old('content') }}</textarea>
        @error('content')
            <div class="invalid-feedback">{{ $message }}</div>
        @enderror
    </div>
    
    <div class="form-group">
        <label for="category_id">Category</label>
        <select id="category_id" name="category_id" class="form-control">
            <option value="">Select a category</option>
            @foreach($categories as $category)
                <option value="{{ $category->id }}" 
                        {{ old('category_id') == $category->id ? 'selected' : '' }}>
                    {{ $category->name }}
                </option>
            @endforeach
        </select>
    </div>
    
    <div class="form-group">
        <label for="image">Featured Image</label>
        <input type="file" id="image" name="image" class="form-control">
    </div>
    
    <button type="submit" class="btn btn-primary">Create Post</button>
</form>
@endsection
```

## Authentication

### Laravel Breeze/Fortify Setup
```bash
# Install Laravel Breeze
composer require laravel/breeze --dev
php artisan breeze:install
npm install && npm run dev
php artisan migrate

# Or Laravel Fortify
composer require laravel/fortify
php artisan vendor:publish --provider="Laravel\Fortify\FortifyServiceProvider"
php artisan migrate
```

### Authentication Usage
```php
// Controllers
use Illuminate\Support\Facades\Auth;

public function login(Request $request)
{
    $credentials = $request->validate([
        'email' => 'required|email',
        'password' => 'required',
    ]);
    
    if (Auth::attempt($credentials, $request->remember)) {
        $request->session()->regenerate();
        return redirect()->intended('dashboard');
    }
    
    return back()->withErrors([
        'email' => 'The provided credentials do not match our records.',
    ]);
}

public function logout(Request $request)
{
    Auth::logout();
    $request->session()->invalidate();
    $request->session()->regenerateToken();
    return redirect('/');
}

// Middleware
Route::middleware('auth')->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});

// Authorization Policies
// php artisan make:policy PostPolicy --model=Post
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }
    
    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->isAdmin();
    }
}

// Usage in controllers
$this->authorize('update', $post);

// Usage in Blade templates
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan
```

## Database and Migrations

### Migrations
```php
// Create migration
// php artisan make:migration create_posts_table
public function up()
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->string('title');
        $table->string('slug')->unique();
        $table->text('content');
        $table->text('excerpt')->nullable();
        $table->timestamp('published_at')->nullable();
        $table->foreignId('user_id')->constrained()->onDelete('cascade');
        $table->foreignId('category_id')->nullable()->constrained();
        $table->timestamps();
        $table->softDeletes();
        
        $table->index(['published_at', 'user_id']);
    });
}

// Modify table
// php artisan make:migration add_featured_to_posts_table
public function up()
{
    Schema::table('posts', function (Blueprint $table) {
        $table->boolean('is_featured')->default(false)->after('published_at');
    });
}

// Pivot table
// php artisan make:migration create_post_tag_table
public function up()
{
    Schema::create('post_tag', function (Blueprint $table) {
        $table->id();
        $table->foreignId('post_id')->constrained()->onDelete('cascade');
        $table->foreignId('tag_id')->constrained()->onDelete('cascade');
        $table->timestamps();
        
        $table->unique(['post_id', 'tag_id']);
    });
}
```

### Database Seeding
```php
// database/seeders/DatabaseSeeder.php
public function run()
{
    User::factory(10)->create();
    Category::factory(5)->create();
    Post::factory(50)->create();
}

// database/factories/PostFactory.php
class PostFactory extends Factory
{
    public function definition()
    {
        return [
            'title' => $this->faker->sentence(),
            'slug' => $this->faker->slug(),
            'content' => $this->faker->paragraphs(3, true),
            'published_at' => $this->faker->optional()->dateTimeBetween('-1 month', 'now'),
            'user_id' => User::factory(),
            'category_id' => Category::factory(),
        ];
    }
}

// Run seeders
php artisan db:seed
php artisan db:seed --class=PostSeeder
```

## Middleware

### Creating Middleware
```php
// php artisan make:middleware CheckRole
class CheckRole
{
    public function handle(Request $request, Closure $next, string $role)
    {
        if (!auth()->check() || !auth()->user()->hasRole($role)) {
            abort(403, 'Unauthorized action.');
        }
        
        return $next($request);
    }
}

// Register in app/Http/Kernel.php
protected $routeMiddleware = [
    'role' => \App\Http\Middleware\CheckRole::class,
];

// Usage
Route::middleware('role:admin')->group(function () {
    Route::resource('users', UserController::class);
});

Route::get('/admin', [AdminController::class, 'index'])
    ->middleware('role:admin');
```

## Artisan Commands

### Common Commands
```bash
# Application
php artisan serve
php artisan key:generate
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Database
php artisan migrate
php artisan migrate:rollback
php artisan migrate:refresh
php artisan db:seed

# Make commands
php artisan make:controller PostController
php artisan make:model Post -mcr  # model, migration, controller, resource
php artisan make:middleware CheckAge
php artisan make:request StorePostRequest
php artisan make:policy PostPolicy
php artisan make:factory PostFactory
php artisan make:seeder PostSeeder
php artisan make:command SendEmails

# Queue
php artisan queue:work
php artisan queue:retry all

# Cache
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

### Custom Commands
```php
// php artisan make:command SendEmails
class SendEmails extends Command
{
    protected $signature = 'email:send {user} {--queue}';
    protected $description = 'Send emails to users';
    
    public function handle()
    {
        $userId = $this->argument('user');
        $queue = $this->option('queue');
        
        $this->info('Sending emails...');
        
        if ($queue) {
            $this->info('Emails queued for processing.');
        } else {
            $this->info('Emails sent immediately.');
        }
    }
}
```

## Testing

### Feature Tests
```php
// tests/Feature/PostTest.php
class PostTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_user_can_view_posts()
    {
        $posts = Post::factory(3)->create();
        
        $response = $this->get('/posts');
        
        $response->assertStatus(200);
        $response->assertSee($posts[0]->title);
    }
    
    public function test_authenticated_user_can_create_post()
    {
        $user = User::factory()->create();
        $category = Category::factory()->create();
        
        $response = $this->actingAs($user)->post('/posts', [
            'title' => 'Test Post',
            'content' => 'Test content',
            'category_id' => $category->id,
        ]);
        
        $response->assertRedirect('/posts');
        $this->assertDatabaseHas('posts', [
            'title' => 'Test Post',
            'user_id' => $user->id,
        ]);
    }
    
    public function test_guest_cannot_create_post()
    {
        $response = $this->post('/posts', [
            'title' => 'Test Post',
            'content' => 'Test content',
        ]);
        
        $response->assertRedirect('/login');
    }
}
```

## Deployment

### Production Setup
```bash
# Optimize application
php artisan config:cache
php artisan route:cache
php artisan view:cache
composer install --optimize-autoloader --no-dev

# Set proper permissions
chmod -R 755 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```

### Environment Configuration
```env
# Production .env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://yourdomain.com

DB_CONNECTION=mysql
DB_HOST=your-host
DB_PORT=3306
DB_DATABASE=your-database
DB_USERNAME=your-username
DB_PASSWORD=your-password

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

MAIL_MAILER=smtp
MAIL_HOST=your-mail-host
MAIL_PORT=587
```

---

## Common Patterns

| Pattern | Use Case | Implementation |
|---------|----------|----------------|
| Repository Pattern | Data access abstraction | Interface + implementation |
| Observer Pattern | Model events | Event listeners |
| Strategy Pattern | Different algorithms | Service classes |
| Factory Pattern | Object creation | Factories and builders |

## Laravel Packages

| Package | Purpose | Installation |
|---------|---------|--------------|
| Laravel Debugbar | Development debugging | `composer require barryvdh/laravel-debugbar --dev` |
| Laravel Telescope | Application monitoring | `composer require laravel/telescope` |
| Spatie Permission | Role and permission management | `composer require spatie/laravel-permission` |
| Laravel Horizon | Queue monitoring | `composer require laravel/horizon` |

---

## Resources
- [Official Laravel Documentation](https://laravel.com/docs)
- [Laravel News](https://laravel-news.com)
- [Laracasts](https://laracasts.com)
- [Laravel Daily](https://laraveldaily.com)
- [Awesome Laravel](https://github.com/chiraggude/awesome-laravel)

---
*Originally compiled from various sources. Contributions welcome!*