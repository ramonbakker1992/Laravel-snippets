# Laravel Snippets

## FormRequest

**Improvements**
- Easy to use attribute manipulation before validation
- Policy is required by default. If no policy is defined, an error will be thrown.
- Rules are defined on the model. As well if validation and authorization are required for the corresponding model.

**Code**
```
<?php

namespace App\App\Foundation;

use Illuminate\Foundation\Http\FormRequest as BaseFormRequest;

class FormRequest extends BaseFormRequest
{
    /**
     * The attributes that should be modified before passing it to the validator.
     *
     * @var array
     */
    public function beforeValidation()
    {
        return [
            //
        ];
    }

    /**
     * Modify the attributes as defined in beforeValidation() before validation
     *
     * @return \Illuminate\Contracts\Validation\Validator
     */
    protected function getValidatorInstance()
    {
        $this->merge($this->beforeValidation());
        
        return parent::getValidatorInstance();
    }

    /**
     * The rules that should apply on the request which are defined in the model
     * 
     * @return array
     */
    public function rules()
    {
        return $this->getModel()->rules();
    }

    /**
     * Determine if the request is authorized.
     * Authorization should be handled in policies.
     *
     * @return bool
     */
    public function authorize()
    {
        if ($this->getModel()->authorization && is_null(policy(get_class($this->getModel())))) {
            throw new Exception('There are no policies set for '.get_class($this->getModel()).'. You should set the policies in AuthServiceProvider.php');
        }

        return true;
    }
}
```
**Usage:**
```
<?php

namespace App\Http\{namespace};

use App\App\Foundation\FormRequest;
use App\Domain\{namespace};

class {Model}Request extends FormRequest
{
    /**
     * Return a new model instance
     *
     * @return Model
     */
    public function getModel()
    {
        return new {Model};
    }
}
```

## FormRequest

**Improvements**
- Validation is required by default. If no validation rules are set, an error will be thrown.
- Validation rules can be set per request type (post, put/patch, delete)
- Mass assignment protection disabled by default. Only `id` property is guarded.
- All date and datetime columns will be automaticly casted to a Carbon instance.
- The default connection will be set automaticly. When using relations on multiple different connections, laravel does'nt know which connection to use by default and needs to be explicitly defined on the model.

**Code**
```
<?php

namespace App\App\Foundation;

use Illuminate\Database\Eloquent\Model as BaseModel;

class Model extends BaseModel
{
    /**
     * Construct the Model instance and setup the properties
     * 
     * @return void
     */
    public function __construct()
    {
        $this->casts = $this->getCastableDateColumns();
        $this->connection = config('database.default');
    }

    /**
     * The attributes that are not mass assignable.
     *
     * @var array
     */
    protected $guarded = [
        'id',
    ];

    /**
     * Determine if validation is required.
     *
     * @var bool
     */
    public $validation = true;

    /**
     * Determine if authorization is required.
     *
     * @var bool
     */
    public $authorization = true;

    /**
     * The rules that should apply on the request.
     * This method is staticly beïng called from within the FormRequest
     * 
     * @return array
     */
    public static function rules() 
    {
        $self = new static;

        if ($self->validation && empty($self->validation())) {
            throw new Exception('There are no validation rules set for '.get_class($self).'. Validation can be disabled by overwriting the $validation property.');
        }

        return $self->validation();
    }

    /**
     * The validation rules that should apply on the request
     * If no rule seperation (store/update) is required, the validation method 
     * can be overwritten so that it returns the validation rules.
     * 
     * @return array
     */
    public function validation()
    {
        $request = request();

        if ($request->method() == 'POST') 
        {
            return array_merge(
                $this->defaultValidation(), 
                $this->storeValidation()
            );
        } 
        elseif ($request->method() == 'PUT' || $request->method() == 'PATCH') 
        {
            return array_merge(
                $this->defaultValidation(), 
                $this->updateValidation()
            );
        } 
        elseif ($request->method() == 'DELETE') 
        {
            return $this->deleteValidation();
        }
    }

    /**
     * The validation rules that should apply on both POST and PUT/PATCH requests
     * Can be used when separating validation logic based on the request type (POST, PUT/PATCH or DELETE)
     * If no seperation is required, the validation() method can be overwritten.
     *
     * @return array
     */
    public function defaultValidation()
    {
        return [];
    }

    /**
     * The validation rules that should only apply on store (POST) requests
     *
     * @return array
     */
    public function storeValidation()
    {
        return [];
    }

    /**
     * The validation rules that should only apply on update (PUT/PATCH) requests
     *
     * @return array
     */
    public function updateValidation()
    {
        return [];
    }

    /**
     * The validation rules that should only apply on delete (DELETE) requests
     *
     * @return array
     */
    public function deleteValidation()
    {
        return [];
    }

    public function getCastableDateColumns()
    {
        $tableColumns = \DB::getDoctrineSchemaManager()->listTableColumns($this->getTable());
        
        $castableDateTypes = [
            'Doctrine\DBAL\Types\DateTimeType' => 'datetime',
            'Doctrine\DBAL\Types\DateType' => 'date',
        ];

        $castableDateColumns = [];

        foreach ($tableColumns as $column) {
            if (in_array(get_class($column->getType()), array_keys($castableDateTypes))) {
                $castableDateColumns[$column->getName()] = $castableDateTypes[get_class($column->getType())];
            }
        }

        return $castableDateColumns;
    }
}
```
**Usage:**
```
<?php

namespace App\Domain\{namespace};

use App\App\Foundation\Model;

class {Model} extends Model
{
    public function validation()
    {
        return [
            //
        ];
    }
}
```
