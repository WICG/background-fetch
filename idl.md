# Getting/creating background fetches

```
partial interface ServiceWorkerRegistration {
  readonly attribute BackgroundFetchManager backgroundFetch;
};

[Exposed=(Window,Worker)]
interface BackgroundFetchManager {
  Promise<BackgroundFetchRegistration> fetch(DOMString tag, (RequestInfo or sequence<RequestInfo>) requests);
  Promise<BackgroundFetchRegistration?> getPending(DOMString tag);
  Promise<sequence<BackgroundFetchRegistration>> getAllPending();
};

[Exposed=(Window,Worker)]
interface BackgroundFetchRegistration {
  readonly attribute DOMString tag;
  readonly attribute FrozenArray<Request> requests;

  void abort();
};
```

# Reacting to success/failure

```
partial interface ServiceWorkerGlobalScope {
  attribute EventHandler onbackgroundfetch;
  attribute EventHandler onbackgroundfetcherror;
  attribute EventHandler onbackgroundfetchabort;
};

[Constructor(DOMString type, BackgroundFetchEventInit init), Exposed=ServiceWorker]
interface BackgroundFetchEvent : ExtendableEvent {
  readonly attribute DOMString tag;
};

dictionary BackgroundFetchEventInit : ExtendableEventInit {
  required DOMString tag;
};

[Constructor(DOMString type, BackgroundFetchResultsEventInit init), Exposed=ServiceWorker]
interface BackgroundFetchResultsEvent : BackgroundFetchEvent {
  readonly attribute maplike<Request, Response> fetches;
};

dictionary BackgroundFetchResultsEventInit : BackgroundFetchEventInit {
  required maplike<Request, Response> fetches;
};
```
