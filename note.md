### Step1 

Must to git clone submodules to checkout the [book theme](https://github.com/alex-shpak/hugo-book)

```
git clone https://github.com/hjlarry/hjlarry.github.io.git --recurse-submodules
```

### Step2

For layout partials modify, add the code below to `themes/book/layouts/partials/docs/inject/footer.html`

```
<br>
<div style="text-align: center;font-size:xx-small;">
    Powered by <a href="https://gohugo.io" target="_blank">Hugo</a> | Theme by <a href="https://github.com/alex-shpak/hugo-book" target="_blank">hugo-book</a>
</div>
```