# Laravel-Breeze-for-Blade-with-Multi-User-Login-Authentication-Guide
CREATE MULTI USER IN LARAVEL BREEZE

First you need to install larel breeze

composer require laravel/breeze --dev
php artisan breeze:install
php artisan migrate
npm install

Step 1: Add Role to the users table migration "database/migrations/xxxx_xx_xx_000000_users_table.php".

    $table->enum('role', ['admin', 'seller', 'customer'])->default('customer'); //change to your prefered roles


Step 2: Add role to the User Model "app/Models/User.php".

    protected $fillable = [
        'role', // Add this
    ];


    // Add this also

    public function isAdmin(): bool
    {
        return $this->role === 'admin';
    }
    public function isSeller(): bool
    {
        return $this->role === 'seller';
    }
    public function isCustomer(): bool
    {
        return $this->role === 'customer';
    }


Step 3: Edit the public function store of RegisteredUserController.php "app/Http/Controller/Auth/RegisteredUserController"

    public function store(Request $request): RedirectResponse
    {
        $request->validate([
            'role' => ['required', 'string', 'in:admin,seller,customer'], // add this
            'fname' => ['required', 'string', 'max:255'],
            'mname' => ['nullable', 'string', 'max:255'],
            'lname' => ['required', 'string', 'max:255'],
            'email' => ['required', 'string', 'lowercase', 'email', 'max:255', 'unique:'.User::class],
            'password' => ['required', 'confirmed', Rules\Password::defaults()],
        ]);

        $user = User::create([
            'role' => $request->role, // add this
            'fname' => $request->fname,
            'mname' => $request->mname,
            'lname' => $request->lname,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);

        event(new Registered($user));

	// Uncomment this if you want auto login
        // Auth::login($user);

        return redirect(route('login', absolute: false));
    }

Step 4: Edit the public function store of AuthenticatedSessionController "app/Http/Controllers/Auth/AuthenticatedSessionController".

    public function store(LoginRequest $request): RedirectResponse
    {
        $request->authenticate();

        $request->session()->regenerate();

        $user = Auth::user();

        if ($user->role === 'admin') {
            return redirect()->route('admin.dashboard');
        } elseif ($user->role === 'seller') {
            return redirect()->route('seller.dashboard');
        } else {
            return redirect()->route('customer.dashboard');
        }
    }


Step 5: Create a Role Middleware.
	
     command:

     php artisan make:middleware Role // This will create Role.php in "app/Http/Middleware" folder

     // Open the Role.php and paste this code

     
     <?php

     namespace App\Http\Middleware;

     use Closure;
     use Illuminate\Http\Request;
     use Symfony\Component\HttpFoundation\Response;

     class Role
     {
         /**
          * Handle an incoming request.
          *
          * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
          */
         public function handle(Request $request, Closure $next, string $role): Response
         {
             if (! $request->user() || $request->user()->role !== $role) {
                 abort(403, 'Unauthorized access.');
             }

             return $next($request);
         }
     }

     // OR Use this one if you want to make to share route the same route for users
     For Example: Route::middleware(['auth', 'role:admin,seller'])->group(function () {
    			Route::get('/sales/reports', [SalesReportController::class, 'index'])->name("sales.reports");
		});

     <?php

     namespace App\Http\Middleware;

     use Closure;
     use Illuminate\Http\Request;
     use Symfony\Component\HttpFoundation\Response;

     class RoleMiddleware
     {
         /**
          * Handle an incoming request.
          *
          * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
          */
         public function handle(Request $request, Closure $next, ...$roles): Response
         {
              if (! $request->user() || !in_array($request->user()->role, $roles)) {
                 abort(403, 'Unauthorized access.');
             }
             return $next($request);
         }
     }


Step 6: Edit the app.php "bootstrap/app.php" and add the middleware alias of role
      
      <?php

      use Illuminate\Foundation\Application;
      use Illuminate\Foundation\Configuration\Exceptions;
      use Illuminate\Foundation\Configuration\Middleware;

      return Application::configure(basePath: dirname(__DIR__))
          ->withRouting(
              web: __DIR__.'/../routes/web.php',
              commands: __DIR__.'/../routes/console.php',
              health: '/up',
          )
          ->withMiddleware(function (Middleware $middleware): void {
              $middleware->alias([
                  'role' => App\Http\Middleware\Role::class,
              ]); // add this code
          })
          ->withExceptions(function (Exceptions $exceptions): void {
              //
          })->create();
     

Step 7: Create the dashboards for multi-user in "resources/views" folder

	// For Admin
	resources/views/admin/dashboard.blade.php
	// For Seller
	reresources/views/seller/dashboard.blade.php
	// For Customer
	resources/views/customer/dashboard.blade.php

Step 8: Create the controllers for the multi-user dashboards

	// For Admin

	php artisan make:controller Admin/DashboardController
	
	// Paste this code in DashboardController.php "app/Http/Controllers/Admin/Dashboard.php
	
		<?php

		namespace App\Http\Controllers\Admin;

		use App\Http\Controllers\Controller;
		use Illuminate\Http\Request;

		class DashboardController extends Controller
		{
    		     public function index()
    		     {
        	         return view('admin.dashboard');
    		     }
		}

	// For Seller

	php artisan make:controller Seller/DashboardController

	// Paste this code in DashboardController.php "app/Http/Controllers/Seller/Dashboard.php
	
		<?php

		namespace App\Http\Controllers\Seller;

		use App\Http\Controllers\Controller;
		use Illuminate\Http\Request;

		class DashboardController extends Controller
		{
    		     public function index()
    		     {
        	         return view('seller.dashboard');
    		     }
		}


	// For Customer

	php artisan make:controller Customer/DashboardController

	// Paste this code in DashboardController.php "app/Http/Controllers/Customer/Dashboard.php
	
		<?php

		namespace App\Http\Controllers\Customer;

		use App\Http\Controllers\Controller;
		use Illuminate\Http\Request;

		class DashboardController extends Controller
		{
    		     public function index()
    		     {
        	         return view('customer.dashboard');
    		     }
		}


Step 9: Edit the Routes in web.php 

	<?php

	use App\Http\Controllers\ProfileController;
	use Illuminate\Support\Facades\Route;

	use App\Http\Controllers\Admin\DashboardController as AdminDashboardController;
	use App\Http\Controllers\Seller\DashboardController as SellerDashboardController;
	use App\Http\Controllers\Customer\DashboardController as CustomerDashboardController;

	Route::get('/', function () {
	    return view('welcome');
	});

	Route::middleware('auth')->group(function () {
	    Route::get('/profile', [ProfileController::class, 'edit'])->name('profile.edit');
	    Route::patch('/profile', [ProfileController::class, 'update'])->name('profile.update');
	    Route::delete('/profile', [ProfileController::class, 'destroy'])->name('profile.destroy');
	});

	// Admin Routes
	Route::middleware(['auth', 'role:admin'])->group(function () {
	    Route::get('/admin/dashboard', [AdminDashboardController::class, 'index'])->name('admin.dashboard'); // Admin Dashboard
	});

	// Seller Routes
	Route::middleware(['auth', 'role:seller'])->group(function () {
	    Route::get('/seller/dashboard', [SellerDashboardController::class, 'index'])->name('seller.dashboard'); // Seller Dashboard
	});

	// Customer Routes
	Route::middleware(['auth', 'role:customer'])->group(function () {
	    Route::get('/customer/dashboard', [CustomerDashboardController::class, 'index'])->name('customer.dashboard'); // Customer Dashboard
	});
	require __DIR__.'/auth.php';


Final Step: Migrate and Serve then Test

	php artisan migrate:fresh
	npm run build
	php artisan serve

