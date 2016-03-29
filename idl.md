# Creating background cache operations

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

# Reacting to success/failure

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
