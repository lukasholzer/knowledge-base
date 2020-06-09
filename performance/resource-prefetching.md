# Speed up Web Performance with Resource Prefetching

All the examples below have to be added in the `<head>` section of the DOM.

## DNS prefetching

Browser can start DNS resolution as quickly as possible

```html
<link rel="dns-prefetch" href="//other-domain.com">
```

## Pre-Connect

Pre-Connect is similar to the dns-prefetch but includes the TCP handshake and optional TLS negotiations.

```html
<link rel="preconnect" href="https://other-domain.com">
```

## Pre-Render

Downloads all assets from the specified webpage.

> This could be very dangerous only use if it is really needed!

```html
<link rel="prerender" href="http://other-domain.com">
```


## Pre-load

Preloads a certain resource that might be referenced through javascript in an earlier point of time.
Can even be uses for fonts and images

Link to the [MDN Preload] reference.

```html
<link rel="preload" href="main.js" as="script">
```


[MDN Preload](https://developer.mozilla.org/en-US/docs/Web/HTML/Preloading_content)