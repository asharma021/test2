$i = 0
Get-Content .gitlab-ci.yml | ForEach-Object {
    $i++
    if ($i -ge 35 -and $i -le 60) {
        "{0,4}: {1}" -f $i, $_
    }
}
