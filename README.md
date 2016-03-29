This is a proposal for a background-caching API, to handle large upload/downloads in the background.

# The problem

The service worker is capable of fetching and caching assets, the size of which is unrestricted. However, if the user navigates away from the site or closes the browser, the service worker is likely to be killed. This can happen even if there's a pending promise passed to `extendableEvent.waitUntil`, if it hasn't resolved within a few minutes the browser may consider it an abuse of service worker and kill the process.

This makes it difficult to download and cache large assets such as podcasts and movies. Even if the service worker isn't killed, having to keep the service worker and therefore the browser in memory during this potentially long operation is wasteful.

# Goals

* Enable background-caching of multiple resources
* Enable background-uploading of multiple resources
* Allow the OS to handle the fetch, so the browser doesn't need to continue running
* Allow the OS to show UI to indicate the progress of the fetch, and perhaps allow the user to pause/abort
* Allow the OS to deal with poor connectivity by pausing/resuming the download/upload
* Allow the app to react to success/failure of the fetch, perhaps by showing a notification

# API sketch

[Check out the IDL](idl.md) if you're into that kind of thing.

## Requesting and managing background downloads

```js
navigator.serviceWorker.ready
  .then(reg => reg.bgCache.register(cacheName, requests))
  .then(bgCacheReg => …);
```

The above shouldn't require user-permission as the operation is user-visible and cancellable. Although `.register` may reject if the user has explicitly disabled background activity, or the operating system has (eg battery saving mode).

The operation is similar to `cache.addAll` in that it's atomic, and considers `!response.ok` to be a failure.

The `bgCacheReg` instance has a few properties:

```js
bgCacheReg.cacheName; // the name of the cacheName
bgCacheReg.requests;  // an array of requests being processed
bgCacheReg.done;      // a promise for operation success/failure
bgCacheReg.abort();   // abort the operation
```

You can get all the pending operations using:

```js
navigator.serviceWorker.ready
  .then(reg => reg.bgCache.getRegistrations());
```

## Reacting to success/failure

This is done via events in the service worker:

```js
self.addEventListener('bgcache', event => {
  event.waitUntil(); // extend the event
  event.cacheName;   // name of the cache that was populated
  event.requests;    // requests that were fetched and cached
});

self.addEventListener('bgcacheerror', event => {
  // event is same as above, but .requests are requests that failed to cache
});

self.addEventListener('bgcacheabort', event => {
  // event is same as bgcacheerror
});
```

# Examples

## Downloading a podcast in reaction to a push message

```js
self.addEventListener('push', event => {
  if (event.data.text() == 'new-podcasts') {
    event.waitUntil(
      getUrlsForNewPodcasts().then(urls => {
        return Promise.all(
          urls.map(url => self.registration.bgCache.register('podcasts', url))
        );
      })
    );
  }
});

self.addEventListener('bgcache', event => {
  if (event.cacheName == 'podcasts')) {
    self.registration.showNotification("You have new podcasts!");
  }
});

self.addEventListener('bgcacheerror', event => {
  if (event.cacheName == 'podcasts')) {
    self.registration.showNotification("Failed to download " + event.requests[0].url);
  }
});
```

## Downloading a level of a game

```js
// in the page:
fetch('/level-20-assets.json').then(r => r.json()).then(data => {
  return navigator.serviceWorker.ready
    .then(reg => reg.bgCache.register('level-20', data.urls))
    .then(bgCacheReg => bgCacheReg.done)
    .catch(() => caches.open('level-20').then(cache => cache.addAll(data.urls)))
    .then(() => updateUIToShowLevel20AsReady());
  });
});
```

## Uploading a video in the background

We would need to allow the cache API to be able to store non-GET requests for this to work.

```js
// in the page:
form.addEventListener('submit', event => {
  event.preventDefault();
  const body = new FormData(form);
  const request = new Request('/upload-video', { body });

  navigator.serviceWorker.ready
    .then(reg => {
      return reg.bgCache.register('completed-uploads', request)
        .then(bgCacheReg => bgCacheReg.done)
        .catch(() => fetch('/upload-video', { body }))
        .then(() => showUploadAsComplete())
        // hide notifications - we don't need them if the page is still open
        .then(() => reg.getNotifications({ tag: 'upload-complete' }))
        .then(notifications => notifications[0] && notifications[0].close())
    })
});
```

```js
// in the service worker:
self.addEventListener('bgcache', event => {
  if (event.cacheName == 'completed-uploads')) {
    self.registration.showNotification("Photo uploaded!");
    event.waitUntil(
      caches.open(event.cacheName).then(c => c.delete(event.requests[0]))
    );
  }
});

self.addEventListener('bgcacheerror', event => {
  if (event.cacheName == 'completed-uploads')) {
    self.registration.showNotification("Photo upload failed");
    event.waitUntil(
      event.requests[0].blob().then(blob => addToIndexedDBOrSomething(blob))
    );
  }
});
```

# Relation to one-off background sync

There's some overlap, and background-cache may be an alternative for some of the simpler background sync use-cases, but mostly they'll be used together. For example, background sync could be used to process an outbox, but background-cache could be used within the `sync` handler for individual messages that have large attachments.

Background-cache is intended to be very user-visible, as such it doesn't really make sense for non-massive transfers such as IM messages.

# Future ideas

I think the above is a reasonable MVP, although `BackgroundCacheRegistration` could be extended with high-level niceties like properties/events to indicate state (uploading/downloading/paused) and progress.
