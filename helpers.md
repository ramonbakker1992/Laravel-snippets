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
