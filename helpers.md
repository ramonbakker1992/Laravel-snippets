### Easy Form Helpers
```php
/**
 * Replaces html array notation in array dot notation
 *
 * Example:
 * "book[author][name]" becomes "book.author.name"
 *
 * @param  string  $path
 * @return string
 */
function html2dot($path) {
    return str_replace(['[',']'], ['.',''], $path);
}

/**
 * Replaces array dot notation in html array notation
 *
 * Example:
 * "book.author.name" becomes "book[author][name]"
 *
 * @param  string  $path
 * @return string
 */
function dot2html($path) {
    $arr = explode('.', $path); $first = $arr[0]; unset($arr[0]);
    return count($arr) ? $first.'['.implode('][', $arr) .']' : $first;
}

/**
 * Access an object property using "dot" notation
 *
 * @param  object  $object
 * @param  string|null  $path
 * @param  mixed  $default
 * @return mixed
 */
function xobject_get(object $object, $path, $default = null) {
    return array_reduce(explode('.', $path), function ($o, $p) use ($default) { 
        return is_numeric($p) ? $o[$p] ?? $default : $o->$p ?? $default; 
    }, $object);
}

/**
 * Access an array's property using "dot" notation
 *
 * @param  array  $array
 * @param  string|null  $path
 * @param  mixed  $default
 * @return mixed
 */
function xarray_get(array $array, $path, $default = null) {
    return array_reduce(explode('.', $path), function ($a, $p) use ($default) { 
        return $a[$p] ?? $default; 
    }, $array);
}

/**
 * Replaces placeholders from a string with object or array values using "dot" notation
 *
 * Example:
 * "The book {title} was written by {author.name}" becomes "The book Harry Potter was written by J.K. Rowling"
 *
 * @param  array|object  $data
 * @param  string  $template
 * @return string
 */
function render_template($data, string $template) {
    preg_match_all("/\{([^\}]*)\}/", $template, $matches); 
    $replace = [];
    foreach ($matches[1] as $param) { 
        $replace['{'.$param.'}'] = is_object($data) ? xobject_get($data, $param) : xarray_get($data, $param); 
    }
    return strtr($template, $replace);
}
```
### Auditable Trait
```php
<?php

namespace App\Traits;

use App\Models\AuditLog;
use Illuminate\Database\Eloquent\Model;

trait Auditable
{
    public static function bootAuditable()
    {
        static::created(function (Model $model) {
            self::audit('created', $model);
        });

        static::updated(function (Model $model) {
            self::audit('updated', $model);
        });

        static::deleted(function (Model $model) {
            self::audit('deleted', $model);
        });
    }

    protected static function audit($description, $model)
    {
        AuditLog::create([
            'description'  => $description,
            'subject_id'   => $model->id ?? null,
            'subject_type' => get_class($model) ?? null,
            'user_id'      => auth()->id() ?? null,
            'properties'   => $model ?? null,
            'host'         => request()->ip() ?? null,
        ]);
    }
}
```
### MultiTenantModel Trait
```php
<?php

namespace App\Traits;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

trait MultiTenantModelTrait
{
    public static function bootMultiTenantModelTrait()
    {
        if (!app()->runningInConsole() && auth()->check()) {
            $isAdmin = auth()->user()->roles->contains(1);
            static::creating(function ($model) use ($isAdmin) {
// Prevent admin from setting his own id - admin entries are global.

// If required, remove the surrounding IF condition and admins will act as users
                if (!$isAdmin) {
                    $model->team_id = auth()->user()->team_id;
                }
            });
            if (!$isAdmin) {
                static::addGlobalScope('team_id', function (Builder $builder) {
                    $field = sprintf('%s.%s', $builder->getQuery()->from, 'team_id');

                    $builder->where($field, auth()->user()->team_id)->orWhereNull($field);
                });
            }
        }
    }
}
```
### Notification example
```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class TeamMemberInvite extends Notification
{
    use Queueable;

    protected $url;

    public function __construct($url)
    {
        $this->url = $url;
    }

    public function via($notifiable)
    {
        return ['mail'];
    }

    public function toMail($notifiable)
    {
        return $this->getMessage();
    }

    public function getMessage()
    {
        return (new MailMessage)
            ->subject(config('app.name') . ': invitation ')
            ->greeting('Hi,')
            ->line('We invite you to join our team!')
            ->line('Please click the link bellow.')
            ->action('Register', $this->url)
            ->line('Thank you')
            ->line(config('app.name') . ' Team')
            ->salutation(' ');
    }
```
### MediaUpload Trait
```php
<?php

namespace App\Http\Controllers\Traits;

use Illuminate\Http\Request;

trait MediaUploadingTrait
{
    public function storeMedia(Request $request)
    {
// Validates file size
        if (request()->has('size')) {
            $this->validate(request(), [
                'file' => 'max:' . request()->input('size') * 1024,
            ]);
        }

// If width or height is preset - we are validating it as an image
        if (request()->has('width') || request()->has('height')) {
            $this->validate(request(), [
                'file' => sprintf(
                    'image|dimensions:max_width=%s,max_height=%s',
                    request()->input('width', 100000),
                    request()->input('height', 100000)
                ),
            ]);
        }

        $path = storage_path('tmp/uploads');

        try {
            if (!file_exists($path)) {
                mkdir($path, 0755, true);
            }
        } catch (\Exception $e) {
        }

        $file = $request->file('file');

        $name = uniqid() . '_' . trim($file->getClientOriginalName());

        $file->move($path, $name);

        return response()->json([
            'name'          => $name,
            'original_name' => $file->getClientOriginalName(),
        ]);
    }
}
```
### API Controller example
```php
<?php

namespace App\Http\Controllers\Api\V1\Admin;

use App\Http\Controllers\Controller;
use App\Http\Controllers\Traits\MediaUploadingTrait;
use App\Http\Requests\StoreTaskRequest;
use App\Http\Requests\UpdateTaskRequest;
use App\Http\Resources\Admin\TaskResource;
use App\Models\Task;
use Gate;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class TaskApiController extends Controller
{
    use MediaUploadingTrait;

    public function index()
    {
        abort_if(Gate::denies('task_access'), Response::HTTP_FORBIDDEN, '403 Forbidden');

        return new TaskResource(Task::all());
    }

    public function store(StoreTaskRequest $request)
    {
        $task = Task::create($request->all());

        if ($request->input('attachments', false)) {
            $task->addMedia(storage_path('tmp/uploads/' . basename($request->input('attachments'))))->toMediaCollection('attachments');
        }

        return (new TaskResource($task))
            ->response()
            ->setStatusCode(Response::HTTP_CREATED);
    }

    public function show(Task $task)
    {
        abort_if(Gate::denies('task_show'), Response::HTTP_FORBIDDEN, '403 Forbidden');

        return new TaskResource($task);
    }

    public function update(UpdateTaskRequest $request, Task $task)
    {
        $task->update($request->all());

        if ($request->input('attachments', false)) {
            if (!$task->attachments || $request->input('attachments') !== $task->attachments->file_name) {
                if ($task->attachments) {
                    $task->attachments->delete();
                }

                $task->addMedia(storage_path('tmp/uploads/' . basename($request->input('attachments'))))->toMediaCollection('attachments');
            }
        } elseif ($task->attachments) {
            $task->attachments->delete();
        }

        return (new TaskResource($task))
            ->response()
            ->setStatusCode(Response::HTTP_ACCEPTED);
    }

    public function destroy(Task $task)
    {
        abort_if(Gate::denies('task_delete'), Response::HTTP_FORBIDDEN, '403 Forbidden');

        $task->delete();

        return response(null, Response::HTTP_NO_CONTENT);
    }
}
```
### Gates Define Middleware
```php
<?php

namespace App\Http\Middleware;

use App\Models\Role;
use Closure;
use Illuminate\Support\Facades\Gate;

class AuthGates
{
    public function handle($request, Closure $next)
    {
        $user = \Auth::user();

        if ($user) {
            $roles            = Role::with('permissions')->get();
            $permissionsArray = [];

            foreach ($roles as $role) {
                foreach ($role->permissions as $permissions) {
                    $permissionsArray[$permissions->title][] = $role->id;
                }
            }

            foreach ($permissionsArray as $title => $roles) {
                Gate::define($title, function ($user) use ($roles) {
                    return count(array_intersect($user->roles->pluck('id')->toArray(), $roles)) > 0;
                });
            }
        }

        return $next($request);
    }
}
```
### SetLocale Middleware
```php
<?php

namespace App\Http\Middleware;

use Closure;

class SetLocale
{
    public function handle($request, Closure $next)
    {
        if (request('change_language')) {
            session()->put('language', request('change_language'));
            $language = request('change_language');
        } elseif (session('language')) {
            $language = session('language');
        } elseif (config('panel.primary_language')) {
            $language = config('panel.primary_language');
        }

        if (isset($language)) {
            app()->setLocale($language);
        }

        return $next($request);
    }
}
```
### User Model Example (with some handy features)
```php
<?php

namespace App\Models;

use App\Notifications\VerifyUserNotification;
use Carbon\Carbon;
use Hash;
use Illuminate\Auth\Notifications\ResetPassword;
use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Support\Str;
use \DateTimeInterface;

class User extends Authenticatable
{
    use SoftDeletes, Notifiable, HasFactory;

    public $table = 'users';

    protected $hidden = [
        'remember_token',
        'password',
    ];

    protected $dates = [
        'email_verified_at',
        'created_at',
        'updated_at',
        'deleted_at',
    ];

    protected $fillable = [
        'name',
        'email',
        'email_verified_at',
        'password',
        'remember_token',
        'created_at',
        'updated_at',
        'deleted_at',
        'team_id',
    ];

    protected function serializeDate(DateTimeInterface $date)
    {
        return $date->format('Y-m-d H:i:s');
    }

    public function getIsAdminAttribute()
    {
        return $this->roles()->where('id', 1)->exists();
    }

    public function __construct(array $attributes = [])
    {
        parent::__construct($attributes);
        self::created(function (User $user) {
            $registrationRole = config('panel.registration_default_role');

            if (!$user->roles()->get()->contains($registrationRole)) {
                $user->roles()->attach($registrationRole);
            }
        });
    }

    public function getEmailVerifiedAtAttribute($value)
    {
        return $value ? Carbon::createFromFormat('Y-m-d H:i:s', $value)->format(config('panel.date_format') . ' ' . config('panel.time_format')) : null;
    }

    public function setEmailVerifiedAtAttribute($value)
    {
        $this->attributes['email_verified_at'] = $value ? Carbon::createFromFormat(config('panel.date_format') . ' ' . config('panel.time_format'), $value)->format('Y-m-d H:i:s') : null;
    }

    public function setPasswordAttribute($input)
    {
        if ($input) {
            $this->attributes['password'] = app('hash')->needsRehash($input) ? Hash::make($input) : $input;
        }
    }

    public function sendPasswordResetNotification($token)
    {
        $this->notify(new ResetPassword($token));
    }

    public function roles()
    {
        return $this->belongsToMany(Role::class);
    }

    public function team()
    {
        return $this->belongsTo(Team::class, 'team_id');
    }
}
```
### Custom config example
```php
<?php

return [
    'date_format'               => 'd-m-Y',
    'time_format'               => 'H:i:s',
    'primary_language'          => 'nl',
    'available_languages'       => [
        'nl' => 'Dutch',
    ],
    'registration_default_role' => '2',
];
```
### x
```php
//
```
### x
```php
//
```
### x
```php
//
```
### x
```php
//
```
### x
```php
//
```
