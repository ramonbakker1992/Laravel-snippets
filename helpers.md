```php
/**
 * Replaces html array notation in array dot notation
 *
 * Example:
 * "book[author][name]" becomes "book.author.name"
 *
 * @param  string
 * @return string
 */
function html2dot($html) {
    return str_replace(['[',']'], ['.',''], $html);
}

/**
 * Replaces array dot notation in html array notation
 *
 * Example:
 * "book.author.name" becomes "book[author][name]"
 *
 * @param  string
 * @return string
 */
function dot2html($dot) {
    $arr = explode('.', $dot); $first = $arr[0]; unset($arr[0]);
    return count($arr) ? $first.'['.implode('][', $arr) .']' : $first;
}

/**
 * Access an object property by dot notation
 * Note; Function replaced by laravel's build-in helper object_get()
 *
 * @param  object
 * @param  string
 * @return mixed
 */
// function object_get(object $object, string $path) {
//     return array_reduce(explode('.', $path), function ($o, $p) { return $o->$p; }, $object);
// }

/**
 * Replaces placeholders from a string with object values
 *
 * Example:
 * "The book {title} was written by {author.name}" becomes "The book Harry Potter was written by J.K. Rowling"
 *
 * @param  object
 * @param  string
 * @return string
 */
function renderTemplate(object $object, string $template) {
    preg_match_all("/\{([^\}]*)\}/", $template, $matches); 
    $replace = [];
    foreach ($matches[1] as $param) { $replace['{'.$param.'}'] = object_get($object, $param); }
    return strtr($template, $replace);
}
```
