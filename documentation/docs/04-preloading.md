---
title: Loading
---

Page components can define a `load` function that runs before the component is created. The values it returns are passed as props to the page.

`load` functions are called when a page is loaded and are typically used to load data that the page depends on - hence its name. This avoids the user seeing the page update as it loads, as is typically the case with client-side loading.

`load` is the SvelteKit equivalent to `getInitialProps` in Next.js or `asyncData` in Nuxt.js.

Note that `load` will run both on the server side and on the client side. It may therefore not reference any APIs only present in the browser.

The following code shows how to load a blog post and pass it to the page in the `article` prop:

```html
<script context="module">
	import * as api from '$common/api.js';

	export async function load({ page, session }) {
		const { slug } = page.params;

		const res = await api.get(`blog/${slug}.json`);
		const article = await res.json();

		return { article };
	}
</script>

<script>
	export let article;
</script>

<h1>{article.title}</h1>
```

The [routing section](docs#Routing) describes how the dynamic parameter `slug` works.

It should be defined in a `context="module"` script since it is not part of the component instance itself – it runs before the component has been created. See the [tutorial](https://svelte.dev/tutorial/module-exports) for more on the module context.

### Arguments

The `load` function receives two arguments — `page` and `session`.

`page` is a `{ host, path, params, query }` object where `host` is the URL's host, `path` is its pathname, `params` is derived from `path` and the route filename, and `query` is an object of values in the query string.

So if the example above was `src/routes/blog/[slug].svelte` and the URL was `/blog/some-post?foo=bar&baz`, the following would be true:

* `page.path === '/blog/some-post'`
* `page.params.slug === 'some-post'`
* `page.query.foo === 'bar'`
* `page.query.baz === true`

`session` can be used to pass data from the server related to the current request, e.g. the current user. By default it is `undefined`. [Seeding session data](docs#Seeding_session_data) describes how to add data to it.


#### api

In browsers, you can use `fetch` to make AJAX requests, for getting data from your server routes (among other things). On the server it's a little trickier — you can make HTTP requests, but you must specify an origin, and you don't have access to cookies. This means that it's impossible to request data based on the user's session, such as data that requires you to be logged in.

To fix this, SvelteKit provides `api` methods, which work on the server as well as in the client:

```html
<script context="module">
	export async function load() {
		const res = await api.get(`server-route.json`);

		// ...
	}
</script>
```

It is important to note that `load` may run on either the server or in the client browser. Code called inside `load` blocks:
  - should run on the same domain as any upstream API servers requiring credentials
  - should not reference `window`, `document` or any browser-specific objects
  - should not reference any API keys or secrets, which will be exposed to the client


### Return values

If you return a Promise from `load`, the page will delay rendering until the promise resolves. You can also return a plain object. In both cases, the values in the object will be passed into the components as props.

When Sapper renders a page on the server, it will attempt to serialize the resolved value (using [devalue](https://github.com/Rich-Harris/devalue)) and include it on the page, so that the client doesn't also need to call `preload` upon initialization. Serialization will fail if the value includes functions or custom classes (cyclical and repeated references are fine, as are built-ins like `Date`, `Map`, `Set` and `RegExp`).


#### error

If the user navigated to `/blog/some-invalid-slug`, we would want to render a 404 Not Found page. We can do that with `this.error`:

```html
<script context="module">
	export async function load({ page, session }) {
		const { slug } = page.params;

		const res = await api.get(`blog/${slug}.json`);

		if (res.status === 200) {
			const article = await res.json();
			return { props: { article } };
		}

		return { status: 404, message: 'Not found' };
	}
</script>
```

The same applies to other error codes you might encounter.


#### redirect

You can abort rendering and redirect to a different location with `this.redirect`:

```html
<script context="module">
	export async function load({ page, session }) {
		const { user } = session;

		if (!user) {
			return { redirect: { status: 302, to: '/login' } };
		}

		return { props: { user } };
	}
</script>
```