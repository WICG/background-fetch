This is a proposal for a background-fetching API, to handle large upload/downloads in the background.

# The problem

The service worker is capable of fetching and caching assets, the size of which is restricted only by [origin storage](https://storage.spec.whatwg.org/#usage-and-quota). However, if the user navigates away from the site or closes the browser, the service worker is likely to be killed. This can happen even if there's a pending promise passed to `extendableEvent.waitUntil`, if it hasn't resolved within a few minutes the browser may consider it an abuse of service worker and kill the process.

This makes it difficult to download and cache large assets such as podcasts and movies, and upload video and images. Even if the service worker isn't killed, having to keep the service worker and therefore the browser in memory during this potentially long operation is wasteful.

# Goals

* Enable background-download/upload of multiple resources
* Allow the OS to handle the fetch, so the browser doesn't need to continue running
* Allow the OS to show UI to indicate the progress of the fetch, and perhaps allow the user to pause/abort
* Allow the OS to deal with poor connectivity by pausing/resuming the download/upload (may be tricky with uploads, as ranged uploads aren't a thing)
* Allow the app to react to success/failure of the fetch, perhaps by showing a notification

# API sketch

[Check out the IDL](idl.md) if you're into that kind of thing.

## Requesting and managing background downloads

```js
navigator.serviceWorker.ready
  .then(reg => reg.backgroundFetch.fetch(tag, requests))
  .then(bgFetchJob => …);
```

The above shouldn't require user-permission as the operation is user-visible and cancellable.

`backgroundFetch.fetch` will reject if there's already a pending job named `tag`, or if the OS is disallowing background activity (user preference or something like battery saving mode).

The `bgFetchJob` instance looks like:

```js
bgFetchJob.tag;      // a tag string
bgFetchJob.requests; // an array of requests being processed
bgFetchJob.abort();  // abort the operation
```

You can get a pending job using:

```js
navigator.serviceWorker.ready
  .then(reg => reg.backgroundFetch.getPending(tag));
```

…which resolves with a `bgFetchJob`. Or: 

```js
navigator.serviceWorker.ready
  .then(reg => reg.backgroundFetch.getAllPending());
```

…which resolve to an array of `bgFetchJob`s.

## Reacting to success/failure

You can react to success using the following service worker event:

```js
self.addEventListener('backgroundfetch', event => {
  event.waitUntil(); // extend the event
  event.tag;         // tag string
  event.fetches;     // a map of request,response
});
```

It's unclear whether `!response.ok` should be considered success or failure.

```js
self.addEventListener('backgroundfetcherror', event => {
  event.waitUntil(); // extend the event
  event.tag;         // tag string
  event.fetches;     // a map of request,response
});

self.addEventListener('backgroundfetchabort', event => {
  event.waitUntil(); // extend the event
  event.tag;         // tag string
});
```

# Examples

## Downloading a podcast in reaction to a push message

```js
// in the service worker:
self.addEventListener('push', event => {
  if (event.data.text() == 'new-podcasts') {
    event.waitUntil(
      getUrlsForNewPodcasts().then(urls => {
        return Promise.all(
          // using map because we *don't* want all of these to be an atomic action
          urls.map(url => self.registration.backgroundFetch.register('podcast-' + url, url))
        );
      }).catch(() => {
        // Can't fetch urls or background download, just tell the user
        // there are new podcasts
        self.registration.showNotification("New podcasts ready to download!");
      })
    );
  }
});

self.addEventListener('backgroundfetch', event => {
  if (event.tag.startsWith('podcast-')) {
    event.waitUntil(
      caches.open('podcasts').then(cache => {
        // cache all the requests/responses
        return Promise.all(
          [...event.fetches].map(([req, res]) => cache.put(req, res))
        );
      }).then(() => {
        self.registration.showNotification("You have new podcasts!");
      })
    );
  }
});

self.addEventListener('backgroundfetcherror', event => {
  if (event.tag.startsWith('podcast-')) {
    const failedUrl = [...event.fetches.keys()][0].url;
    self.registration.showNotification("Failed to download " + failedUrl);
  }
});
```

## Downloading a level of a game

```js
// in the page:
fetch('/level-20-assets.json').then(r => r.json()).then(data => {
  const tag = 'level-20';

  return navigator.serviceWorker.ready
    .then(reg => reg.backgroundFetch.fetch(tag, data.urls))
    .then(() => backgroundFetchDone(tag))
    // can't background fetch? Download from the page
    .catch(() => {
      return caches.open(tag)
        .then(cache => cache.addAll(data.urls))
    })
    .then(() => updateUIToShowLevelReady(20))
    .catch(() => updateUIToShowLevelFailed(20));
  });
});

function backgroundFetchDone(tag) {
  return new Promise((resolve, reject) => {
    const channel = new BroadcastChannel('background-fetch');
    channel.onmessage = event => {
      if (event.data.tag != tag) return;
      if (event.data.ok) {
        resolve();
      }
      else {
        reject();
      }
    }
    channel.close();
  });
}
```

```js
// in the service worker:
self.addEventListener('backgroundfetch', event => {
  if (event.tag.startsWith('level-')) {
    event.waitUntil(
      caches.open(event.tag).then(cache => {
        // cache all the requests/responses
        return Promise.all(
          [...event.fetches].map(([req, res]) => cache.put(req, res))
        );
      }).then(() => {
        // tell the page
        new BroadcastChannel('background-fetch').postMessage({
          tag: event.tag,
          ok: true
        });
      })
    );
  }
});

self.addEventListener('backgroundfetcherror', event => {
  if (event.tag.startsWith('level-')) {
    new BroadcastChannel('background-fetch').postMessage({
      tag: event.tag,
      ok: false
    });
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
  const tag = 'video-upload-' + videoName;
  const request = new Request('/upload-video', { body, method: 'POST' });

  const reg = await navigator.serviceWorker.ready;

  try {
    // try fetching in the background or from the page
    try {
      await reg.backgroundFetch.fetch(tag, request);
      await backgroundFetchDone(tag);
    }
    catch () {
      // Failed, try and upload from the page.
      // First store the video in IDB in case the user closes the tab
      await storeInIDB(body);
      const response = await fetch('/upload-video', { body, method: 'POST' });
      if (!response.ok) throw Error('Upload failed');
    }

    showUploadAsComplete();
    deleteFromIDB(body);
  }
  catch() {
    showUploadAsFailed();
  }
});

function backgroundFetchDone(tag) {
  return new Promise((resolve, reject) => {
    const channel = new BroadcastChannel('background-fetch');
    channel.onmessage = event => {
      if (event.data.tag != tag) return;
      if (event.data.ok) {
        resolve();
      }
      else {
        reject();
      }
    }
    channel.close();
  });
}
```

```js
// in the service worker:
self.addEventListener('backgroundfetch', event => {
  if (event.tag.startsWith('video-upload-')) {
    self.registration.showNotification("Photo uploaded!");
    new BroadcastChannel('background-fetch').postMessage({
      tag: event.tag,
      ok: true
    });
  }
});

self.addEventListener('backgroundfetcherror', event => {
  if (event.tag.startsWith('video-upload-')) {
    self.registration.showNotification("Photo upload failed");
    new BroadcastChannel('background-fetch').postMessage({
      tag: event.tag,
      ok: true
    });

    // store video in IDB for later retry
    event.waitUntil(
      [...event.fetches.keys()][0].formData()
        .then(body => storeInIDB(body))
    );
  }
});
```

# Relation to one-off background sync

Background-fetch is intended to be very user-visible, via OS-level UI such as a persistent notification, as such background-sync remains a better option for non-massive transfers such as IM messages.

# Future ideas

* A way to get progress of the up/download
* A way to read a response mid-download (eg, start playing a downloading podcast)
