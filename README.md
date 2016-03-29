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
* Doesn't require user-permission, as it's user-visible and cancellable, although may reject if user has explicitly disabled background activity

# API sketch

Creating background cache operations:

```
partial interface ServiceWorkerRegistration {
  readonly attribute BackgroundCacheManager bgCache;
};

[Exposed=(Window,Worker)]
interface BackgroundCacheManager {
  Promise<BackgroundCacheRegistration> register(DOMString cacheName, (RequestInfo or sequence<RequestInfo>) requests);
  Promise<sequence<BackgroundCacheRegistration>> getRegistrations();
};

[Exposed=(Window,Worker)]
interface BackgroundCacheRegistration {
  readonly attribute DOMString cacheName;
  readonly attribute sequence<Request> requests;
  readonly attribute Promise<any> done;

  void abort();
};
```

Reacting to success/failure:

```
partial interface ServiceWorkerGlobalScope {
  attribute EventHandler onbgcache;
  attribute EventHandler onbgcacheerror;
  attribute EventHandler onbgcacheabort;
};

[Constructor(DOMString type, BackgroundCacheEventInit init), Exposed=ServiceWorker]
interface BackgroundCacheEvent : ExtendableEvent {
  readonly attribute DOMString cacheName;
  readonly attribute sequence<Request> requests;
};

dictionary BackgroundCacheEventInit : ExtendableEventInit {
  required DOMString cacheName;
  required sequence<Request> requests;
};
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
