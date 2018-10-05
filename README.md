This is a proposal for a background-fetching API, to handle large upload/downloads without requiring an origin's clients (pages/workers) to be running throughout the job.

# The problem

A service worker is capable of fetching and caching assets, the size of which is restricted only by [origin storage](https://storage.spec.whatwg.org/#usage-and-quota). However, if the user navigates away from the site or closes the browser, the service worker is likely to be killed. This can happen even if there's a pending promise passed to `extendableEvent.waitUntil` - if it hasn't resolved within a few minutes the browser may consider it an abuse of service worker and kill the process.

This is excellent for battery and privacy, but it makes it difficult to download and cache large assets such as podcasts and movies, and upload video and images.

This spec aims to solve the long-running fetch case, without impacting battery live or privacy beyond that of a long-running download.

# Features

* Allow fetches (requests & responses) to continue even if the user closes all windows & workers to the origin.
* Allow a single job to involve many requests, as defined by the app.
* Allow the browser/OS to show UI to indicate the progress of that job, and allow the user to pause/abort.
* Allow the browser/OS to deal with poor connectivity by pausing/resuming the download.
* Allow the app to react to success/failure of the job, perhaps by caching the results.
* Allow access to background-fetched resources as they fetch.

# API Design

## Starting a background fetch

```js
const registration = await navigator.serviceWorker.ready;
const bgFetchReg = await registration.backgroundFetch.fetch(id, requests, options);
```

* `id` - a unique identifier for this background fetch.
* `requests` - a sequence of URLs or `Request` objects.
* `options` - an object containing any of:
  * `icons` - A sequence of icon definitions.
  * `title` - Something descriptive to show in UI, such as "Uploading 'Holiday in Rome'" or "Downloading 'Catastrophe season 2 episode 1'".
  * `downloadTotal` - The total unencoded download size. This allows the UI to tell the user how big the total of the resources is. If omitted, the UI will be in a non-determinate state.

`backgroundFetch.fetch` will reject if:

* The user does not want background downloads, which may be a origin/browser/OS level setting.
* If there's already a registered background fetch job associated with `registration` identified by `id`.
* Any of the requests have mode `no-cors`.
* The browser fails to store the requests and their bodies.
* `downloadTotal` suggests there isn't enough quota to complete the job.

The operation will later fail if:

* A fetch with a non-`GET` request fails. There's no HTTP mechanism to resume uploads.
* Fetch rejects when the user isn't offline. As in, CORS failure, MIX issues, CSP issue etc etc.
* The unencoded download size exceeds the provided `downloadTotal`.
* The server provides an unexpected partial response.
* Quota is exceeded.
* A response does not have an ok status.

If `downloadTotal` is exceeded, the operation fails immediately. Otherwise, the other fetches will be given a chance to settle. This means if the user is uploading 100 photos, 99 won't be aborted just because one fails. The operation as a whole will still be considered a failure, but the app can communicate what happened to the user.

`bgFetchReg` has the following:

* `id` - identifier string.
* `uploadTotal` - total bytes to send.
* `uploaded` - bytes sent so far.
* `downloadTotal` - as provided.
* `downloaded` - bytes stored so far.
* `result` -  `""`, `"success"`, `"failure"`.
* `failureReason` - `""`, `"aborted"`, `"bad-status"`, `"fetch-error"`, `"quota-exceeded"`, `"download-total-exceeded"`.
* `recordsAvailable` - Does the underlying request/response data still exist? It's removed once the operation is complete.
* `activeFetches` - provides access to the in-progress fetches.
* `onprogress` - Event when the above properties change.
* `abort()` - abort the whole background fetch job. This returns a promise that resolves with a boolean, which is true if the operation successfully aborted.
* `match(request, options)` - Access one of the fetch records.
* `matchAll(request, options)` - Access some of the fetch records.


## Getting an instance of a background fetch

```js
const registration = await navigator.serviceWorker.ready;
const bgFetchReg = await registration.backgroundFetch.get(id);
```

If no job with the identifier `id` exists, `get` resolves with undefined.

## Getting all background fetches

```js
const registration = await navigator.serviceWorker.ready;
const ids = await registration.backgroundFetch.getIds();
```

…where `ids` is a sequence of unique identifier strings.

## Background fetch records

```js
const bgFetchReg = await registration.backgroundFetch.get(id);
const record = bgFetchReg.match(request);
```

`record` has the following:

* `request`. A Request.
* `responseReady`. A promise for a Response. This will reject if the fetch fails.

## Reacting to success

Fires in the service worker if all responses in a background fetch were successfully & fully read, and all status codes were [ok](https://fetch.spec.whatwg.org/#ok-status).

```js
addEventListener('backgroundfetchsuccess', bgFetchEvent => {
  // …
});
```

`bgFetchEvent` extends [`ExtendableEvent`](https://w3c.github.io/ServiceWorker/#extendableevent), with the following additional members:

* `registration` - The background fetch registration.
* `updateUI({ title, icons })` - update the UI, eg "Uploaded 'Holiday in Rome'", "Downloaded 'Catastrophe season 2 episode 1'", or "Level 5 ready to play!".

Once this event is fired, the background fetch job is no longer stored against the registration, so `backgroundFetch.get(bgFetchEvent.id)` will resolve with undefined.

Once this has completed (including promises passed to `waitUntil`), `recordsAvailable` becomes false, and the requests/responses can no longer be accessed.

## Reacting to failure

As `backgroundfetchsuccess`, but one or more of the fetches encountered an error.

```js
addEventListener('backgroundfetchfail', bgFetchEvent => {
  // …
});
```

Aside from the event name, the details are the same as `backgroundfetchsuccess`.

## Reacting to abort

If a background fetch job is aborted, either by the user, or by the developer calling `abort()` on the background fetch job, the following event is fired in the service worker:

```js
addEventListener('backgroundfetchabort', bgFetchAbortEvent => {
  // …
});
```

`bgFetchAbortEvent` extends [`ExtendableEvent`](https://w3c.github.io/ServiceWorker/#extendableevent), with the following additional members:

* `registration` - The background fetch registration.

The rest is as `backgroundfetchsuccess`.

## Reacting to click

If the UI representing a background fetch job is clicked, either during or after the job, the following event is fired in the service worker:

```js
addEventListener('backgroundfetchclick', bgFetchClickEvent => {
  // …
});
```

* `registration` - The background fetch registration.

Since this is a user interaction event, developers can call `clients.openWindow` in response.

The rest is as `backgroundfetchsuccess`.

# Possible UI

Background fetches will be immediately visible using a UI of the browser's choosing. On Android this is likely to be a sticky notification displaying:

* The origin of the site.
* The chosen icon.
* The title.
* A progress bar.
* Buttons to pause/abort the job.
* Potentially a way to prevent the origin starting any more background downloads.

If aborted by the user, the notification is likely to disappear immediately. If aborted via code, the notification will remain in an "ended" state.

Once ended, the progress bar will be replaced with an indication of how much data was transferred. The button to pause/abort the job will no longer be there. The notification will no longer be sticky.

If the job has ended, clicking the notification may also close/hide it (in addition to firing the event).

# Quota usage

The background fetch requests & in-progress responses can be accessed at any time until the `backgroundfetchsuccess`, `backgroundfetchfail`, or `backgroundfetchabort` event end, so they count against origin quota.

# Lifecycle

The background fetch job is linked to the service worker registration. If the service worker is unregistered, background fetches will be aborted (without firing events) and its storage purged.

This means the feature may be used in "private browsing modes" that use a temporary profile, as the fetches will be cancelled and purged along with the service worker registrations.

# Security & privacy

[Some browsers can already start downloads without user interaction](http://output.jsbin.com/fimukur/quiet), but they're easily abortable. We're following the same pattern here.

Background fetch may happen as the result of other background operations, such as push messages. In this case the background fetch may start in a paused state, effectively asking the user permission to continue.

The icon and title of the background fetch are controllable by the origin. Hopefully the UI can make clear which parts are under the site's control, and which parts are under the browser's control (origin, data used, abort/pause). There's some prior art here with notifications.

Background fetches are limited to CORS, to avoid opaque responses taking up origin quota.

# Relation to one-off background sync

Background-fetch is intended to be very user-visible, via OS-level UI such as a persistent notification, as such background-sync remains a better option for non-massive transfers such as IM messages.

# Examples

## Downloading a movie

Movies are either one large file (+ extra things like metadata and artwork), or 1000s of chunks.

In the page:

```js
downloadButton.addEventListener('click', async () => {
  try {
    const movieData = getMovieDataSomehow();
    const reg = await navigator.serviceWorker.ready;
    const bgFetch = await reg.backgroundFetch.fetch(`movie-${movieData.id}`, movieData.urls, {
      icons: movieData.icons,
      title: `Downloading ${movieData.title}`,
      downloadTotal: movieData.downloadTotal
    });
    // Update the UI.

    bgFetch.addEventListener('progress', () => {
      // Update the UI some more.
    });
  } catch (err) {
    // Display an error to the user
  }
});
```

In the service worker:

```js
addEventListener('backgroundfetchsuccess', (event) => {
  event.waitUntil(async function() {
    // Copy the fetches into a cache:
    try {
      const cache = await caches.open(event.registration.id);
      const records = await event.registration.matchAll();
      const promises = records.map(async (record) => {
        const response = await record.responseReady;
        await cache.put(record.request, response);
      });
      await Promise.all(promises);
      const movieData = await getMovieDataSomehow(event.registration.id);
      await event.updateUI({ title: `${movieData.title} downloaded!` });
    } catch (err) {
      event.updateUI({ title: `Movie download failed` });
    }
  }());
});

// There's a lot of this that's copied from 'backgroundfetchsuccess', but I've avoided
// abstracting it for this example.
addEventListener('backgroundfetchfail', (event) => {
  event.waitUntil(async function() {
    // Store everything successful, maybe we can just refetch the bits that failed
    try {
      const cache = await caches.open(event.registration.id);
      const records = await event.registration.matchAll();
      const promises = records.map(async (record) => {
        const response = await record.responseReady.catch(() => undefined);
        if (response && response.ok) {
          await cache.put(record.request, response);
        }
      });
      await Promise.all(promises);
    } finally {
      const movieData = await getMovieDataSomehow(event.registration.id);
      await event.updateUI({ title: `${movieData.title} download failed.` });
    }
  }());
});

addEventListener('backgroundfetchclick', (event) => {
  event.waitUntil(async function() {
    const movieData = await getMovieDataSomehow(event.registration.id);
    clients.openWindow(movieData.pageUrl);
  }());
});

// Allow the data to be fetched while it's being downloaded:
addEventListener('fetch', (event) => {
  if (isMovieFetch(event)) {
    event.respondWith(async function() {
      const cachedResponse = await caches.match(event.request);
      if (cachedResponse) return cachedResponse;

      // Maybe it's mid-download?
      const movieData = getMovieDataSomehow(event.request);
      const bgFetch = await registration.backgroundFetch.get(`movie-${movieData.id}`);

      if (bgFetch) {
        const record = await bgFetch.match(event.request);
        if (record) return record.responseReady;
      }

      return fetch(event.request);
    }());
  }
  // …
});
```

## Uploading photos

In the page:

```js
uploadButton.addEventListener('click', async () => {
  try {
    // Create the requests:
    const galleryId = createGalleryIdSomehow();
    const photos = getPhotoFilesSomehow();
    const requests = photos.map((photo) => {
      const body = new FormData();
      body.set('gallery', galleryId);
      body.set('photo', photo);

      return new Request('/upload-photo', {
        body,
        method: 'POST',
        credentials: 'include',
      });
    });

    const reg = await navigator.serviceWorker.ready;
    const bgFetch = await reg.backgroundFetch.fetch(`photo-upload-${galleryId}`, requests, {
      icons: getAppIconsSomehow(),
      title: `Uploading photos`,
    });

    // Update the UI.

    bgFetch.addEventListener('progress', () => {
      // Update the UI some more.
    });
  } catch (err) {
    // Display an error to the user
  }
});
```

In the service worker:

```js
addEventListener('backgroundfetchsuccess', (event) => {
  event.waitUntil(async function() {
    const galleryId = getGalleryIdSomehow(event.registration.id);
    await event.updateUI({ title: `Photos uploaded` });

    // The gallery is complete, so we can show it to the user's friends:
    await fetch('/enable-gallery', {
      method: 'POST',
      body: new URLSearchParams({ id: galleryId }),
    })
  }());
});

addEventListener('backgroundfetchfail', (event) => {
  event.waitUntil(async function() {
    const records = await event.registration.matchAll();
    let failed = 0;

    for (const record of records) {
      const response = await record.responseReady.catch(() => undefined);
      if (response && response.ok) continue;
      failed++;
    }

    if (successful) {
      event.updateUI({ title: `${failed}/${records.length} uploads failed` });
    }
  }());
});

addEventListener('backgroundfetchclick', (event) => {
  event.waitUntil(async function() {
    const galleryId = getGalleryIdSomehow(event.registration.id);
    clients.openWindow(`/galleries/${galleryId}`);
  }());
});
```
