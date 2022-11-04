# cra_nginx
Securing create-react-app hosted on nginx

I was setting up a site built with create-react-app and hosting it on a VPS using Debian 11 with nginx as the webserver, and I found some challenges to securing it that I wanted to document the solutions I ended up with. Mostly just for my own benifit in case I needed to do it again, but it might be useful to others too. By no means should this be considered the ideal setup, it is just what I ended up wiht after several days of researching and experimenting.

The first thing I did was purchage a certificate. This was straightforward and there are plenty of good resources available to assist with this.

The first thing I wanted to do was make sure that the certificate was setup correctly. Inside the nginx config file for my site, this is what I ended up with for the ssl settings:

```
  ssl_buffer_size 4k;
  ssl_certificate /path/to/some.crt;
  ssl_certificate_key /path/to/some.key;
  ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
  ssl_ecdh_curve secp521r1:secp384r1;
  ssl_protocols TLSv1.3 TLSv1.2;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;
  ssl_stapling on;
  ssl_stapling_verify on;
```

This setup yields an A+ rating at the Qualys SSL Server Test.

The next thing was to add recommended response headers, I will leave the `Content-Security-Policy` out for now and go over it in more detail later. Here is the headers added to the nginx config file for my site:

```
  add_header Permissions-Policy "fullscreen=(self)";
  add_header Referrer-Policy "strict-origin";
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-XSS-Protection "1";
```

I won't go in to the details of thsese, there are lots of explanations available, but this is the set I landed on. 

When it came to `Content-Security-Policy`, there is a lot information available for setting it up with `create-react-app`, most of the information suggests using various tools that will add the CSP to a meta tag in the index.html file that CRA creates. The issue with this is that it does not provide all the available directives. 

- frame-ancestors
- report-uri
- report-to
- sandbox

So although using the meta tag approach provides a good amount of protection, using a response header is more ideal. 

One of the biggest challenges to adding a CSP header with nginx is being able to use the `script-src` directive with `strict-dynamic` using a nonce that is populated in all the script tags in the `index.html` file dynamically by the server for each request. This challenge seems to be why most of the information I found suggested using the meta tag method. But it seems that using the meta tag has another limitation that can be overcome by setting a nonce value on the server for each request. The methods to add CSP via a meta tag generate a nonce value at build time and inject it into the script tags then. Since it uses the same nonce each time, it is open to attack. Other suggestions don't use `strict-dynamic` and nonces at a all, they just set `script-src` to `self` which ensures scripts are only loaded from the site itself, but there are known vulnerabilities with this.

Setting up nginx to inject a unique nonce with each response first required setting up a variable `$csp` that that had the unique value that would then be used to set the header. Here is the section of the nginx config file for my site:

```
 set $csp "default-src 'none';
           base-uri 'none';
           img-src 'self';
           form-action 'none';
           frame-ancestors 'self';
           frame-src 'self';
           object-src 'none';
           require-trusted-types-for 'script;
           script-src 'nonce-$request_id' 'strict-dynamic' 'unsafe-inline' https:;
           style-src 'self';";
```

In this you can see the `script-src` had `nonce-$request_id`, `$request_id` is predefinded in nginx and is a unique value generated for each request, so works nicely here. `unsafe-inline` and `https:` are added to support old browsers that don't honor `strict-dynamic`.

The next thing is to make nginx do a replacement on the script tags in the index.html to add the same value of the `$request_d` to the `nonce` attribute.

```
  sub_filter_once off;
  sub_filter "##NONCE##" $request_id;
```

For this to work the version of nginx you are using must have been built to support `--with-http_sub_module`. You can see if yours does with `nginx -V`. This will look for `##NONCE##` in the index.html and replace it with the value of `$request_id`. The version of nginx included with Debian 11 has it enabled out of the box.

The next thing was to get the index.html to have an attribute on the script tags like this: `nonce="##NONCE##"`. Lots of information said that adding `__webpack_nonce__ = "##NONCE##;` as the first line of the entry file index.js would inject `##NONCE##` to all the script tags that webpack injected into index.html when running `npm run build` to create the build to be deployed to the server. This did not work, I am not sure why exactly, if it is someting that CRA is doing, or if it is no longer supported by webpack. 

What I eventaully did, just to avoid having to `eject` CRA, was add a custom script to my project.json as `postbuild` so that it would inject the nonce attribute to the script tags in the index.html after the build was completed. There is likely much better ways to do this, but it works.

Here is my script `nonce.js`:

```
const {promises: fsPromises} = require('fs');

async function replaceInFile(filename, replacement) {
  try {
    const contents = await fsPromises.readFile(filename, 'utf-8');
    const replaced = contents.replace(/<script "/g, replacement);

    await fsPromises.writeFile(filename, replaced);
    console.log("nonce added to script");
  } catch (err) {
    console.log(err);
  }
}

replaceInFile('build/index.html', '<script nonce="##NONCE##" ');
```

Here is how it looks in package.json:

```
"build": "react-scripts build",
"postbuild": "node scripts/nonce.js",
```

These are all the pieces it took to get an A+ on the Google CSP Evaluator. Here is my full nginx config file for my site:

```
server {
        listen 443 ssl;
        
        gzip on;
        gzip_types      text/plain application/xml;
        gzip_proxied    no-cache no-store private expired auth;
        gzip_min_length 1000;
        gunzip on;

        ssl_buffer_size 4k;
        ssl_certificate /path/to/some.crt;
        ssl_certificate_key /path/to/some.key;
        ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_ecdh_curve secp521r1:secp384r1;
        ssl_protocols TLSv1.3 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_stapling on;
        ssl_stapling_verify on;

        set $csp "default-src 'none';
                  base-uri 'none';
                  img-src 'self';
                  form-action 'none';
                  frame-ancestors 'self';
                  frame-src 'self';
                  object-src 'none';
                  require-trusted-types-for 'script;
                  script-src 'nonce-$request_id' 'strict-dynamic' 'unsafe-inline' https:;
                  style-src 'self';";

        root /var/www/example.com/html;
        index index.html index.htm;

        server_name example.com;

        location / {
                add_header Content-Security-Policy "${csp}";
                add_header Permissions-Policy "fullscreen=(self)";
                add_header Referrer-Policy "strict-origin";
                add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
                add_header X-Content-Type-Options "nosniff" always;
                add_header X-Frame-Options "SAMEORIGIN" always;
                add_header X-XSS-Protection "1";

                sub_filter_once off;
                sub_filter "##NONCE##" $request_id;

                try_files $uri /index.html; # this makes react router work
        }
}
```



