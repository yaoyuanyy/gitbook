# gitbook
## one step 
```
$ git checkout gh-pages
$ cd gitbook/
$ gitbook build gitbook_source gitbook_web
```

## two step
```
$ git add .
$ git commit -m"msg"
$ git push
```

waiting for a moment
```
curl GET -X https://yaoyuanyy.github.io/gitbook/gitbook_web
```
