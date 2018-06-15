This is a proposal for a background-fetching API, to handle large upload/downloads without requiring an origin's clients (pages/workers) to be running throughout the job.

# The problem

A service worker is capable of fetching and caching assets, the size of which is restricted only by [origin storage](https://storage.spec.whatwg.org/#usage-and-quota). However, if the user navigates away from the site or closes the browser, the service worker is likely to be killed. This can happen even if there's a pending promise passed to `extendableEvent.waitUntil` - if it hasn't resolved within a few minutes the browser may consider it an abuse of service worker and kill the process.

This makes it difficult to download and cache large assets such as podcasts and movies, and upload video and images. Even if the service worker isn't killed, having to keep the service worker in memory during this potentially long operation is wasteful.

# Features

* Allow fetches (requests & responses) to continue even if the user closes all windows & worker to the origin.
* Allow a single job to involve many requests, as defined by the app.
* Allow the browser/OS to show UI to indicate the progress of the fetch, and allow the user to pause/abort.
* Allow the browser/OS to deal with poor connectivity by pausing/resuming the download/upload (may be tricky with uploads, as ranged uploads aren't standardized)
* Allow the app to react to success/failure of the background fetch group, perhaps by caching the results.
* Allow access to background-fetched resources as they fetch.
* Allow the app to display progress information about a background fetch.
* Allow the app to suggest which connection types the fetch should be restricted to.

These are eventual goals, some features be missing from "v1".

# API Design

## Starting a background fetch

```js
const registration = await navigator.serviceWorker.ready;
const bgFetchJob = await registration.backgroundFetch.fetch(id, requests, options);
```

* `id` - a unique identifier for this job.
* `requests` - a sequence of URLs or `Request` objects.
* `options` - an object containing any of:
  * `icons` - A sequence of icon definitions, similar to [`icons` in the manifest spec](https://w3c.github.io/manifest/#icons-member).
  * `title` - Something descriptive to show in UI, such as "Uploading 'Holiday in Rome'" or "Downloading 'Catastrophe season 2 episode 1'".
  * `downloadTotal` - A hint for the UI, so it can display progress before receiving all `Content-Length` headers, or if any are missing. This is in bytes, before transport compression. This is only a hint, the browser will disregard the value if/once it's found to be incorrect.
  * `networkType` - "auto" or "avoid-cellular", defaults to "auto".
  * `powerStatus` - "auto" or "avoid-draining", defaults to "auto".

`networkType`/`powerStatus` are unlikely to be in "v1". The naming here is also difficult, I took them from [background sync](https://github.com/WICG/BackgroundSync/issues/35).

`backgroundFetch.fetch` will reject if:

* The user does not want background downloads, which may be a origin/browser/OS level setting.
* If there's already a registered background fetch job associated with `registration` identified by `id`.
* Any of the requests have mode `no-cors`.
* The browser fails to store the requests and their bodies.
* `downloadTotal` suggests there isn't enough quota to complete the job.

## Getting an instance of a background fetch

```js
const registration = await navigator.serviceWorker.ready;
const bgFetchJob = await registration.backgroundFetch.get(id);
```

`bgFetchJob` has the following members:

* `id` - identifier string.
* `uploadTotal` - total bytes to send.
* `uploadProgress` - bytes sent so far.
* `downloadTotal` - as provided.
* `downloadProgress` - bytes received so far.
* `activeFetches` - provides access to the in-progress fetches.
* `abort()` - abort the whole background fetch job. This returns a promise that resolves with a boolean, which is true if the operation successfully aborted.
* `onprogress` - Event when `downloadProgress` or `uploadProgress` update.

`BackgroundFetch` objects will also include [fetch controller/observer objects](https://github.com/whatwg/fetch/issues/447) as they land.

If no job with the identifier `id` exists, `get` resolves with undefined.

I don't like the naming of `responseReady` above. Ideas?

## Getting all background fetches

```js
const registration = await navigator.serviceWorker.ready;
const identifiers = await registration.backgroundFetch.getIdentifiers();
```

…where `identifiers` is a sequence of unique identifier strings.

Eventually, `registration.backgroundFetch` will become an async iterator, allowing:

```js
for await (const bgFetchJob of registration.backgroundFetch) {
  // …
}
```

## Reacting to 'success'

Fires if all responses in a background fetch were successfully & fully read, and all status codes were [ok](https://fetch.spec.whatwg.org/#ok-status).

```js
addEventListener('backgroundfetched', bgFetchEvent => {
  // …
});
```

`bgFetchEvent` extends [`ExtendableEvent`](https://w3c.github.io/ServiceWorker/#extendableevent), with the following additional members:

* `id` - identifier of the background fetch job.
* `fetches` - provides access to the fetches.
* `updateUI(title)` - update the title, eg "Uploaded 'Holiday in Rome'", "Downloaded 'Catastrophe season 2 episode 1'", or "Level 5 ready to play!". If this isn't called, the browser may update the title itself.

Once this event is fired, the background fetch job is no longer stored against the registration, so `backgroundFetch.get(bgFetchEvent.id)` will resolve with undefined.

## Reacting to failure

As `backgroundfetched`, but one or more of the fetches encountered an error.

```js
addEventListener('backgroundfetchfail', bgFetchFailEvent => {
  // …
});
```

`bgFetchFailEvent` extends `bgFetchEvent`, with the following additional members:

* `fetches` - provides access to the fetches.

Again, once this event is fired, the background fetch job is no longer stored against the registration, so `backgroundFetch.get(bgFetchEvent.id)` will resolve with undefined.

In an earlier draft, this event and `backgroundfetched` were the same thing, including a property that indicated success/failure. It felt wrong when writing examples, so I split them out.

This event should fire after all fetches have completed or been abandoned. As in, this event *doesn't* fire as soon as a single request fails.

## Reacting to abort

If a background fetch job is aborted, either by the user, or by the developer calling `abort()` on the background fetch job, the following event is fired in the service worker:

```js
addEventListener('backgroundfetchabort', bgFetchAbortEvent => {
  // …
});
```

`bgFetchAbortEvent` extends [`ExtendableEvent`](https://w3c.github.io/ServiceWorker/#extendableevent), with the following additional members:

* `id` - identifier of the background fetch job.

Again, once this event is fired, the background fetch job is no longer stored against the registration, so `backgroundFetch.get(bgFetchAbortEvent.id)` will resolve with undefined.

## Reacting to click

If the UI representing a background fetch job is clicked, either during or after the job, the following event is fired in the service worker:

```js
addEventListener('backgroundfetchclick', bgFetchClickEvent => {
  // …
});
```

* `id` - identifier of the background fetch job.
* `state` - "pending", "succeeded" or "failed".

Since this is a user interaction event, developers can call `clients.openWindow` in response. If a window isn't created/focused, the browser may open a related URL (the root of the origin, or a `start_url` from a related manifest).

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

If the job has ended, clicking the notification may close/hide it.

# Retrying

The browser may retry particular requests in response to network failures. Range requests may be used to resume particular requests.

We need to think about which cases are "retryable", and which indicate terminal failure. Eg, a POST that results in a network failure may be "retryable", but a POST that results in a 403 may not. We also need to define how often a "retryable" request can be retried before failing, and any kind of delay before retrying.

Range requests probably don't make sense for non-GET requests.

Eventually we'd like uploads to be resumable, but there's no standard for that.

# Quota usage

The background fetch job, including its requests & in-progress responses, can be accessed at any time until the `backgroundfetched`, `backgroundfetchfail`, or `backgroundfetchabort` event fires, so they count against origin quota. After the point, the event receives the requests and responses (except in the case of abort) for the final time, once these copies are drained, they're gone.

A requirement is to avoid doubling quota usage when transferring background-fetched items into the cache, unless clones exist.

```js
addEventListener('backgroundfetched', event => {
  event.waitUntil(async function() {
    const cache = await caches.open('my-cache');
    const fetches = await event.fetches.values();
    const promises = fetches.map(({request, response}) => {
      if (response && response.ok) {
        return cache.put(request, response);
      }
    });

    await Promise.all(promises);
  }());
});
```

A user agent could implement this using streams. Given a response body can only be read once, it can drain it from background fetch storage, whilst feeding it into the cache.

# Differences to fetch

Background fetch will behave like `fetch()` (including in terms of CSP), with the following exceptions:

* The [keepalive flag](https://fetch.spec.whatwg.org/#request-keepalive-flag) will be set.
* The browser may, for GET requests, download the resource in chunks using range requests, and use them to build a single response. We need to define how this happens.

# Lifecycle

The background fetch job is linked to the service worker registration. If the service worker is unregistered, background fetches will be aborted (without firing events) and its storage purged.

This means the feature may be used in "private browsing modes" that use a temporary profile, as the fetches will be cancelled and purged along with the service worker registrations.

# Security & privacy

[Some browsers can already start downloads without user interaction](http://output.jsbin.com/fimukur/quiet), but they're easily abortable. We're following the same pattern here.

Background fetch may happen as the result of other background operations, such as push messages, but the download notification will be visible, even after completion. Ideally this will let the user know if a particular origin is behaving badly.

Given that background fetches are pausable and abortable, a browser may choose to start a background fetch in a paused state, and present UI to the user asking if they'd like to continue.

The icon and title of the background fetch are controllable by the origin. Hopefully the UI can make clear which parts are under the site's control, and which parts are under the browser's control (origin, data used, abort/pause). There's some prior art here with notifications.

Background fetches are limited to CORS, to avoid opaque responses taking up origin quota.

# Relation to one-off background sync

Background-fetch is intended to be very user-visible, via OS-level UI such as a persistent notification, as such background-sync remains a better option for non-massive transfers such as IM messages.

# Examples

## Downloading a podcast in reaction to a push message

```js
// in the service worker:
addEventListener('push', event => {
  if (event.data.text() == 'new-podcasts') {
    event.waitUntil(async function() {
      const podcasts = await getNewPodcasts();
      await addPodcastsToDatabase(podcasts);

      try {
        // Each podcast should be a separate background download, but each podcast
        // may contain multiple resources, eg mp3 & artwork.
        const promises = podcasts.map(podcast => {
          return self.registration.backgroundFetch.fetch(`podcast-${podcast.id}`, podcast.urls, {
            icons: podcast.icons,
            title: `Downloading ${podcast.showName} - ${podcast.episodeName}`,
            downloadTotal: podcast.downloadTotal
          });
        });

        await Promise.all(promises);
      }
      catch (err) {
        // Can't fetch urls or background download, just tell the user
        // there are new podcasts
        self.registration.showNotification("New podcasts ready to download!");
      }
    }());
  }
});

// Success!
addEventListener('backgroundfetched', event => {
  if (event.id.startsWith('podcast-')) {
    event.waitUntil(async function() {
      // Get podcast by ID
      const podcast = await getPodcast(/podcast-(.*)$/.exec(event.id)[0]);


      // Cache podcasts
      const cache = await caches.open(event.id);
      const fetches = await event.fetches.values();
      const promises = event.fetches.map(({request, response}) => {
        return cache.put(request, response);
      });

      await Promise.all(promises);
      event.updateUI(`Downloaded ${podcast.showName} - ${podcast.episodeName}`);
    }());
  }
});

// Failed!
addEventListener('backgroundfetchfail', event => {
  if (event.id.startsWith('podcast-')) {
    event.waitUntil(async function() {
      // Get podcast by ID
      const podcast = await getPodcast(/podcast-(.*)$/.exec(event.id)[0]);
      // Aww.
      event.updateUI(`Failed to download ${podcast.showName} - ${podcast.episodeName}`);
    }());
  }
});

// Clicked!
addEventListener('backgroundfetchclick', event => {
  if (event.id.startsWith('podcast-')) {
    event.waitUntil(async function() {
      // Get podcast by ID
      const podcast = await getPodcast(/podcast-(.*)$/.exec(event.id)[0]);

      // Happy to open this url no matter what the state is.
      clients.openWindow(podcast.pageUrl);
    }());
  }
});
```

## Playing a podcast as it's background-fetching

```js
addEventListener('fetch', event => {
  if (isPodcastAudioRequest(event.request)) {
    const podcastId = getPodcastId(event.request);

    event.respondWith(async function() {
      const bgFetchJob = await self.registration.backgroundFetch.get(`podcast-${podcastId}`);

      if (bgFetchJob) {
        // Look for response in fetches
        const activeFetch = await bgFetchJob.activeFetches.match(event.request);
        if (activeFetch) return activeFetch.responseDone;
      }

      // Else fall back to cache or network
      const response = await caches.match(event.request);
      return response || fetch(event.request);
    }());
  }
});
```

## Downloading a level of a game

```js
// in the page:
(async function() {
  const response = await fetch('/level-20-assets.json');
  const data = await response.json();
  const id = 'level-20';

  const reg = await navigator.serviceWorker.ready;

  try {
    await self.registration.backgroundFetch.fetch(id, data.urls, {
      icons: data.icons,
      title: "Download level 20",
      downloadTotal: data.downloadTotal
    });
  }
  catch {
    // Can't background fetch? Fallback to downloading from the page:
    const cache = await caches.open(id);
    await cache.addAll(data.urls);
    showInPageNotification('Level 20 ready to play!');
  }
}());
```

```js
// in the service worker:
self.addEventListener('backgroundfetched', event => {
  if (event.id.startsWith('level-')) {
    const level = /^level-(.*)$/.exec(event.id)[1];

    event.waitUntil(async function() {
      const cache = await caches.open(event.id);
      const fetches = await event.fetches.values();
      const promises = event.fetches.map(({request, response}) => {
        return cache.put(request, response);
      });

      await Promise.all(promises);

      event.updateUI(`Level ${level} ready to play!`);
    }());
  }
});

addEventListener('backgroundfetchfail', event => {
  if (event.id.startsWith('level-')) {
    const level = /^level-(.*)$/.exec(event.id)[1];
    event.updateUI(`Failed to download level ${level}`);
  }
});

addEventListener('backgroundfetchclick', event => {
  if (event.id.startsWith('level-')) {
    clients.openWindow(`/`);
  }
});
```

## Uploading a video in the background

```js
// in the page:
form.addEventListener('submit', async event => {
  event.preventDefault();
  const body = new FormData(form);
  const videoName = body.get('video').name;
  const id = 'video-upload-' + videoName;
  const request = new Request('/upload-video', { body, method: 'POST' });

  const reg = await navigator.serviceWorker.ready;

  try {
    await reg.backgroundFetch.fetch(id, request, {
      title: "Uploading video"
    });
  }
  catch () {
    // Failed, try and upload from the page.
    // First store the video in IDB in case the user closes the tab
    await storeInIDB(body);
    const response = await fetch('/upload-video', { body, method: 'POST' });
    if (response.ok) showUploadAsComplete();
  }
});
```

```js
// in the service worker:
addEventListener('backgroundfetched', event => {
  if (event.id.startsWith('video-upload-')) {
    event.updateUI("Video uploaded!");
  }
});

addEventListener('backgroundfetchfail', event => {
  if (event.id.startsWith('video-upload-')) {
    event.updateUI("Upload failed");

    // Store the data in IDB so it isn't lost:
    event.waitUntil(async function() {
      const fetches = await event.fetches.values();
      const formData = fetches[0].request.formData();
      await storeInIDB(formData);
    }());
  }
});
```
