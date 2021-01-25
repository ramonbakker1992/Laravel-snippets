**Html2Dot**

    // user[name] => user.name
    function html2dot($html) {
        return str_replace(['[',']'], ['.',''], $html);
    }


**Dot2Html**

    // user.name => user[name]
    function dot2html($dot) {
        $arr = explode('.', $dot); $first = $arr[0]; unset($arr[0]);
        return count($arr) ? $first.'['.implode('][', $arr) .']' : $first;
    }
