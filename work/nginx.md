## nginx docs
[offical docs](http://nginx.org/en/docs/)  
[Location Block Selection Algorithms](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms)

## regex in nginx

### How to I get variables from location in nginx?
```
location ~/photos/resize/(\d+)/(\d+) {
  # use $1 for the first \d+ and $2 for the second, and so on.
}
or
location ~/photos/resize/(?<width>(\d+))/(?<height>(\d+)) {
  # so here you can use the $width and the $height variables
}
```
$1 refers to the first capturing group (...). When you added another group it referred to that one instead. You can use a non-capturing group (?:...) instead, or refer to the second capturing group $2