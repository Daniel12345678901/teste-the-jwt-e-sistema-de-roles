# Step 1: Install JWT Authentication Package

**First, we need to install the JWT authentication package:** `composer require tymon/jwt-auth`  
**Publish the JWT Auth configuration:** `php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"`  
**Generate the JWT secret key:** `php artisan jwt:secret`  

##Step 2: Update User Model

**Update the User model to implement JWTSubject:**  
```
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Tymon\JWTAuth\Contracts\JWTSubject;

class User extends Authenticatable implements JWTSubject
{
    use HasFactory;

    protected $fillable = ['name', 'email', 'password', 'role_id'];

    protected $hidden = ['password', 'remember_token'];

    public function role()
    {
        return $this->belongsTo(Role::class);
    }

    public function getJWTIdentifier()
    {
        return $this->getKey();
    }

    public function getJWTCustomClaims()
    {
        return [];
    }
}
```

##Step 3: Create Middleware for Role-Based Access

**Create middleware to check the user's role:** `php artisan make:middleware RoleMiddleware`  
**In the generated middleware file (app/Http/Middleware/RoleMiddleware.php), define the middleware:**  
```
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Tymon\JWTAuth\Facades\JWTAuth;

class RoleMiddleware
{
    public function handle(Request $request, Closure $next, $roleIds)
    {
        $user = JWTAuth::parseToken()->authenticate();

        if (!in_array($user->role_id, explode(',', $roleIds))) {
            return response()->json(['error' => 'Unauthorized'], 403);
        }

        return $next($request);
    }
}
```

##Step 4: Register Middleware

**Register the middleware in app/Http/Kernel.php:**  
```
protected $routeMiddleware = [
    // ...
    'role' => \App\Http\Middleware\RoleMiddleware::class,
];
```

##Step 5: Set Up AuthController

**Create the AuthController that handles user registration and login:** `php artisan make:controller AuthController`  
**In the generated controller file (app/Http/Controllers/AuthController.php), define the methods for registration and login:**  
```
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Validator;
use Tymon\JWTAuth\Facades\JWTAuth;

class AuthController extends Controller
{
    public function register(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:6|confirmed',
            'role_id' => 'required|exists:roles,id',
        ]);

        if ($validator->fails()) {
            return response()->json($validator->errors(), 400);
        }

        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
            'role_id' => $request->role_id,
        ]);

        $token = JWTAuth::fromUser($user);

        return response()->json(['token' => $token], 201);
    }

    public function login(Request $request)
    {
        $credentials = $request->only('email', 'password');

        if (!$token = JWTAuth::attempt($credentials)) {
            return response()->json(['error' => 'Invalid Credentials'], 401);
        }

        return response()->json(['token' => $token]);
    }
}
```

##Step 6: Define Routes

**Define routes in routes/api.php for user registration, login, and protected routes:**
```
use App\Http\Controllers\AuthController;
use App\Http\Controllers\UserController;
use Illuminate\Support\Facades\Route;

Route::post('register', [AuthController::class, 'register']);
Route::post('login', [AuthController::class, 'login']);

Route::middleware(['auth:api'])->group(function () {
    Route::get('users', [UserController::class, 'index']);
    Route::get('users/{id}', [UserController::class, 'show']);
    Route::post('users', [UserController::class, 'store']);
    Route::put('users/{id}', [UserController::class, 'update']);
    Route::delete('users/{id}', [UserController::class, 'destroy']);
    
    Route::middleware(['role:1'])->group(function () {
        // Routes accessible only to admin (role_id = 1)
        Route::get('admin', function () {
            return response()->json(['message' => 'Welcome Admin']);
        });
    });

    Route::middleware(['role:1,2'])->group(function () {
        // Routes accessible to admin and doctor (role_id = 1, 2)
        Route::get('doctor', function () {
            return response()->json(['message' => 'Welcome Doctor or Admin']);
        });
    });

    Route::middleware(['role:1,2,3'])->group(function () {
        // Routes accessible to admin, doctor, and patient (role_id = 1, 2, 3)
        Route::get('patient', function () {
            return response()->json(['message' => 'Welcome Patient, Doctor, or Admin']);
        });
    });
});
```
