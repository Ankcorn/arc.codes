# Static (`arc.static`)

## Automatically build and deploy your static assets

Architect projects support a `public/` directory in the root of your project for static assets. The `public/` directory typically includes static assets such as images, styles, and scripts required in your front-end workflows.

Anything in  `public/` directory is available at `http://localhost:3333/_static/` when running in the sandbox.

For `production` and `staging` environments, Architect can have `staging` and `production` S3 buckets for file syncing from the `public/` folder - they'll be available at `https://yourapi.com/_static` once deployed.

The `arc.static` helper resolves URL paths for your static assets, so you're requesting the right file from every environment.

And if you've enabled fingerprinting, `arc.static` also resolves the unique fingerprinted filename in staging and production environments.


## Provisioning

Given the following `.arc` file:

```arc
@app
static-site

@static
staging my-unique-bucket-staging
production my-unique-bucket
```

Running `npx create` will generate `staging` and `production` S3 buckets.

> ⚠️ Warning: S3 buckets are _globally_ unique to all of AWS so you may have to try a few names.


## Fingerprinting

To enable file fingerprinting, add `fingerprint true` to your `@static` pragma, e.g.:
```
@static
staging my-unique-bucket-staging
production my-unique-bucket
fingerprint true
```


## Working Locally with `public/`

Running `npx sandbox` kicks up a sandbox web server (more here about [working locally](/guides/offline)). The folder `public/` at the root of your project will be mounted at `/_static` when you run the web server with `npx sandbox`.

Any file added to this `public/` folder will be served (along with any HTTP functions you've defined).

Most frontend JavaScript workflows involve some sort of build step, so the `public/` folder is a staging area for those build artifacts (along with whatever else you'd like to use it for, of course).

The simplest possible build script defined in `package.json`:

```json
{
  "build": "cp -r src/shared/client public",
  "start": "npm run build && npx sandbox"
}
```

Running the script defined above with `npm run build` just blindly copies files from `src/shared/client` to `public/`. (This could definitely be enhanced by using a module bundler like [Browserify](http://browserify.org/), [Parcel](https://parceljs.org/) or [Webpack](https://webpack.js.org/) depending on your needs!)

Running `npm start` builds the JS and starts a local web server on `http://localhost:3333` for previewing.


## Deploying `public/`

Running `npx deploy` copies `public/` to the `staging` bucket. (If you want to version these assets in S3, you can [enable that feature](https://docs.aws.amazon.com/AmazonS3/latest/dev/Versioning.html) in the AWS Console.)

Alternately you could consider these build artifacts (which they are) and treat your version control system as the place to manage versions (which it is). 😶

Running `ARC_DEPLOY=production npx deploy` copies `public/` to the production bucket.

And, of course, it would be wise to use both of these S3 buckets as origins for your CDN (we're partial to AWS CloudFront).

> 🏌️‍♀️ Protip: `npx deploy static` will deploy the static assets _only_

> ⛳️ Protip #2: `npx deploy [static] --delete` will remove files from the bucket that are no longer locally present


## Linking

Isolation is key to creating a continuous delivery pipeline. It's good to work on our local machines, deploy to a staging environment, and promote to production with total confidence that the system is only improving. Static assets are no different!

As such, there are three environments you need to be concerned about for addressing your static assets:

- Local:
> `http://localhost:3333/_static/<asset>`
- Staging:
> `https://<staging bucket>.s3.<aws region>.amazonaws.com/<asset>`
- Production:
> `https://<production bucket>.s3.<aws region>.amazonaws.com/<asset>`

This is an example production URL from a testing app:
> `https://arc-testapp-production.us-west-1.s3.amazonaws.com/babybeaver.jpg`


## Calling static URLs

`@architect/functions` bundles a helper function for HTTP route handlers that disambiguates URLs called `arc.static`. It accepts a relative path and returns the URL appropriate for the environment it's being invoked in.

```javascript
// src/http/get-index/index.js
let arc = require('@architect/functions')
let static = arc.static

exports.handler = async function http(req) {
  let body = `
    <!doctype html>
    <html>
      <head>
        <title>This is fun!</title>
        <link rel=stylesheet type=text/css href=${static('/main.css')}>
      </head>
      <body>Hello ƛ</body>
      <script src=${static('main.js')}></script>
    </html>
  `
  return {
    type: 'text/html',
    body
  }
}
```


## Ignoring files from `public/`

You can instruct Architect to ignore files from your `public/` directory with the `ignore` directive, like so:
```
@static
staging my-unique-bucket-staging
production my-unique-bucket
ignore
  zip
  tar
```

This works with a simple string search, so if you ignore `foo`, all filenames containing `foo` (or files with paths containing `foo`) will be ignored.

> By default, Architect ignores `.DS_Store`, `node_modules`, and `readme.md` files


See [the static reference](/reference/static) for more details.

<hr>

## Next: [Single Page Apps](/guides/spa)
# Single Page Apps

## Static and dynamic API endpoints coexisting at the same origin

Architect provides two methods to proxy static assets through API Gateway.
 This means your single page application and API can share the same domain name, session support and database access *without CORS* and *without 3rd party proxies*. 

For this guide we'll use the following `.arc` file:

```arc
@app
spa

@http
get /

@static
staging spa-stage-bukkit
production spa-prod-bukkit
```

> 👉🏽 Note: S3 buckets are global to all of AWS so you may need to try a few different names!

## Proxy Public

Lambda is _very good_ at reading and processing text from S3. To enable the proxy add the following to the root Lambda:

```javascript
// src/http/get-index/index.js
let arc = require('@architect/functions')

exports.handler = arc.proxy.public()
```

Now all static assets in `./public` will be served from the root of your application.

The `arc.proxy.public` function accepts an optional configuration param `spa` which will force loading `index.html` no matter what route is invoked (note however that routes defined in `.arc` will take precedence). 

```javascript
// src/http/get-index/index.js
let arc = require('@architect/functions')

exports.handler = arc.proxy.public({spa:true})
```

Set `{spa:false}`, or omit, if you want the proxy to return a `404` error when a directory or file does not exist. 

> Bonus: when `404.html` is present that file will be returned

### Quirks

- No binary files: images, audio video need to be served via S3 static URLs

## Alias for Pretty URLs

The `arc.proxy.public` accepts an `alias` configuration object for mapping pretty URLs:

```javascript
// src/http/get-index/index.js
let arc = require('@architect/functions')

exports.handler = arc.proxy.public({
  alias: {
    '/home': '/templates/home.html',
  }
})
```

## Proxy Plugins

The `arc.proxy.public` can be further augmented with per filetype transform plugins. Each key in `plugins` is a file extension for processing with an array of transform plugins to run anytime that filetype is matched.

The first use case for this feature is to fix URLs. API Gateway creates ugly URLs by default appending `/staging` and `/production` to the application root. This pain goes away once you setup DNS but setting up static sites is much more complicated because most tools do not expect these paths. 

This demonstrates using proxy plugins to transform all links so they are prefixed with the correct URL. 

```javascript
// src/http/get-index/index.js
let arc = require('@architect/functions')

exports.handler = arc.proxy.public({
  plugins: {
    html: ['@architect/proxy-plugin-html-urls'],
    css: ['@architect/proxy-plugin-css-urls'],
    mjs: ['@architect/proxy-plugin-mjs-urls'],
    md: [
      '@architect/proxy-plugin-md',
      '@architect/proxy-plugin-html-urls'
    ]
  },
  alias: {
    '/': '/home.md',
    '/about': '/about.md',
    '/contact': '/contact.md'
  }
})
```
While not necessary until DNS is set up it's super helpful. Transform plugins open the door to other useful capabilities for authoring dynamic single page apps. 

Architect supports the following transform plugins:

*prototyping*

- `@architect/proxy-plugin-html-urls` adds `/staging` or `/production` to HTML elements
- `@architect/proxy-plugin-css-urls` adds `/staging` or `/production` to CSS `@imports` statements
- `@architect/proxy-plugin-mjs-urls` adds `/staging` or `/production` to JS module `import` statements

*esmodules*

- `@architect/proxy-plugin-bare-imports` map bare imports to fully qualified URLs

*syntax transpilers*

- `@architect/proxy-plugin-jsx/react` transpile JSX into `React` calls
- `@architect/proxy-plugin-jsx/preact` transpile JSX into `h` calls
- `@architect/proxy-plugin-tsx/react` transpile TSX into `React` calls
- `@architect/proxy-plugin-tsx/preact` transpile TSX into `h` calls
- `@architect/proxy-plugin-md` transpile Markdown into HTML
- `@architect/proxy-plugin-sass` transpile SCSS into CSS

*release*

- `@architect/proxy-plugin-html-min` minify HTML
- `@architect/proxy-plugin-css-min` minify CSS 
- `@architect/proxy-plugin-mjs-min` minify JS

## Serverless Site Rendering

Prerendering content is great for performance but sometimes you need complete control of the initial HTML payload. In these cases you can enable `ssr` by giving it a module or function to run whenever `index.html` is requested.  

```javascript
// src/http/get-index/index.js
let arc = require('@architect/functions')
let myRenderFun = require('@architect/views/my-render-fun')

exports.handler = arc.proxy.public({
  
  // inline ssr function into config
  async ssr(req) {
    let headers = {'content-type':'text/html'}
    let body = await myRenderFun({state})
    return {headers, body}
  }
})
```

Or reference a local module:

```javascript
// src/http/get-index/index.js
let arc = require('@architect/functions')

exports.handler = arc.proxy.public({
  ssr: './render'
})
```

Or reference a Node module:

```javascript
// src/http/get-index/index.js
let arc = require('@architect/functions')

exports.handler = arc.proxy.public({
  ssr: '@architect/views/render'
})
```

<hr>

## Next: [Sessions](/guides/sessions)

