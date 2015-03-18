

# Introduction #

The Google Toolbox for Mac HTTP Fetcher is a set of classes to simplify http operations in Cocoa applications.

## When would you use the fetcher? ##

  * When you just want a result from one or more requests without writing much networking code
  * When you want the "standard" behavior for connections, such as redirection handling
  * When you want automatic retry on failures
  * When you want to avoid cookie collisions with Safari and other applications
  * When you may be requesting the same resource repeatedly, and want to minimize network traffic
  * When you want to make many requests of a host, but avoid overwhelming the host with simultaneous requests
  * When you want a convenient log of http requests and responses
  * When you need to set a credential for the http operation

# Using the Fetcher #

## Adding the Fetcher to Your Project ##

Check out the “top-of-trunk” fetcher sources with a Mac terminal window and the command on the [source checkout page](http://code.google.com/p/gtm-http-fetcher/source/checkout).

Only the `GTMHTTPFetcher.h/m` source files are required. The Logging class file is optional but useful for all applications. If your application will make a series of related fetches, the Service and History classes will likely be useful.

| **Source files** | **Purpose** |
|:-----------------|:------------|
| GTMHTTPFetcher.h/m | required |
| GTMHTTPFetcherLogging.h/m | http logging (often helpful) |
| GTMHTTPFetcherService.h/m, GTMHTTPFetchHistory.h/m | history across fetches |
| GTMMIMEDocument.h/m, GTMGatherInputStream.h/m | multipart MIME uploads |
| GTMHTTPUploadFetcher.h/m | [resumable-uploads](http://code.google.com/apis/gdata/docs/resumable_upload.html) |

### ARC Compatibility ###

When the fetcher source files are compiled directly into a project that has ARC enabled, then ARC must be disabled specifically for the fetcher source files.

To disable ARC for source files in Xcode 4, select the project and the target in Xcode. Under the target "Build Phases" tab, expand the Compile Sources build phase, select the library source files, then press Enter to open an edit field, and type  `-fno-objc-arc`  as the compiler flag for those files.

## Usage ##

### Fetching a Request ###

Because the fetcher is implemented as a wrapper on NSURLConnection, it is asynchronous.

The fetcher can be used most simply by adding GTMHTTPFetcher.h/m to your project, and fetching a URL like this:

```
  #import "GTMHTTPFetcher.h"
  
  NSURL *url = [NSURL URLWithString:@"http://www.example.com/"];
  NSURLRequest *request = [NSURLRequest requestWithURL:url];
  GTMHTTPFetcher* myFetcher = [GTMHTTPFetcher fetcherWithRequest:request];
  [myFetcher beginFetchWithDelegate:self
                  didFinishSelector:@selector(myFetcher:finishedWithData:error:)];
```

Each fetcher object is good for "one shot"; do not reuse an instance for a second fetch.

The fetcher may be created auto-released (as shown above with `+fetcherWithRequest:`) so it will release itself after the fetch completion callback has been invoked.  The fetcher retains itself implicitly as long as a connection is pending.

But if you may need to cancel the fetcher prior to completion, allocate the fetcher with `initWithRequest:` and have the delegate release the fetcher in the callback.

Upload data may be optionally specified with the fetcher's `postData` and `postStream` properties.

The callback looks like
```
- (void)myFetcher:(GTMHTTPFetcher *)fetcher finishedWithData:(NSData *)retrievedData error:(NSError *)error {
  if (error != nil) {
    // failed; either an NSURLConnection error occurred, or the server returned
    // a status value of at least 300
    //
    // the NSError domain string for server status errors is kGTMHTTPFetcherStatusDomain
    int status = [error code];
  } else {
    // fetch succeeded
  }
}
```

Typically, your callbacks will have a selector named to indicate what was fetched, like
```
imageFetcher:finishedWithData:error:

settingsFetcher:finishedWithData:error:
```

The delegate is retained by the fetcher until the fetch has completed or has been canceled.

**Note:** Because the fetcher retains the delegate until the fetch has completed or has been cancelled, if a selector callback is used, the delegate will not be deallocated while the fetch is in progress. Invoking the fetcher's `stopFetching` method in the delegate's `dealloc` is therefore not useful.

### Blocks-style Callback ###

The callback may also be supplied as an Objective-C block:
```
  [myFetcher beginFetchWithCompletionHandler:^(NSData *retrievedData, NSError *error) {
    if (error != nil) {
      // status code or network error
    } else {
      // succeeded
    }
  }];
```

Other fetcher delegate methods also have block versions.

### Fetching from Secondary Threads ###

The fetcher, like NSURLConnection upon which it is built and like other Cocoa classes with callbacks, executes its callbacks when the thread's run loop is running. A fetch typically should be started on a thread with a run loop, which most often is the main thread.

As an alternative, starting with iOS 6 and Mac OS X 10.7, applications may use an operation queue for
callbacks, avoiding the requirement for a run loop. For example, the callback may be run on a background queue's thread:
```
fetcher.delegateQueue = [[[NSOperationQueue alloc] init] autorelease];
```
The main thread can also be used to run the callback:
```
fetcher.delegateQueue = [NSOperationQueue mainQueue];
```
The application may re-dispatch from the callbacks and notifications to
the application's preferred dispatch queue:
```
[myFetcher beginFetchWithCompletionHandler:^(NSData *retrievedData, NSError *error) {
  if (error == nil) {
    dispatch_async(myDispatchQueue, ^{
      ...
   });
  }
}];
```

As the fetcher is inherently asynchronous, applications using the fetcher should not create additional threads in order to get asynchronous http networking behavior.

### Background Fetches in iOS Applications ###

On iOS 4 and later, the fetch may optionally continue while the app is sent to the background by the user. The fetch continues until it finishes, or until it is stopped by OS expiration of background execution.

To enable background fetches, set the `shouldFetchInBackground` property of the fetcher to YES.

Background fetches are always enabled in Mac OS X; the property is ignored.

### Passing Values to Callbacks ###

The fetcher provides the method `setProperty:forKey:` and the property `userData` for passing values to the callbacks.

The fetch request is available in the `mutableRequest` property.

### Cookies ###

There are three supported mechanisms for remembering cookies between fetches.

By default, GTMHTTPFetcher uses a mutable array held statically to track cookies for all instantiated fetchers. This keeps cookies set by the server during the fetch from interfering with Safari cookie settings, and vice versa.  The fetcher cookies are forgotten when the application quits.

To rely instead on WebKit's global NSHTTPCookieStorage, call setCookieStorageMethod: with kGTMHTTPFetcherCookieStorageMethodSystemDefault.

If the fetcher is created from a GTMHTTPFetcherService object
then the fetcher is set to use the cookie storage in the
service object rather than the application static storage. Each fetcher service object instance has its own "cookie jar".

### Monitoring Received Data ###

An optional received data selector can be set as the `receivedDataSelector` property. The selector should have the signature

```
- (void)myFetcher:(GTMHTTPFetcher *)fetcher receivedData:(NSData *)dataReceivedSoFar;
```

The number of bytes received so far during the fetch is available as `[dataReceivedSoFar length]`. This number may go down if a redirect causes the download to begin again from a new server.

If supplied by the server, the anticipated total download size is available as`[[myFetcher response] expectedContentLength]` (though it may be -1 for unknown download sizes.)

### Notifications for the Network Activity Indicator ###

The fetcher posts notifications `kGTMHTTPFetcherStartedNotification` and `kGTMHTTPFetcherStoppedNotification` to indicate when its network activity has started and stopped. A started notification is always balanced by a single stopped notification from the same fetcher instance. Your application can observe those notifications in its code to start and stop the network activity indicator.

When the fetcher is paused between retry attempts, it posts `kGTMHTTPFetcherRetryDelayStartedNotification` and `kGTMHTTPFetcherRetryDelayStoppedNotification` notifications. The retry delay periods are between network activity, not during it.

### Proxies ###

Proxy handling is invisible so long as the system has a valid credential in
the keychain, which is normally true (else most NSURL-based apps would encounter proxy errors.)

But when there is a proxy authetication error, the the fetcher
will invoke the callback method with the NSURLChallenge in the error's
userInfo. The error method can get the challenge info like this:
```
 NSURLAuthenticationChallenge *challenge = [[error userInfo] objectForKey:kGTMHTTPFetcherErrorChallengeKey];
 BOOL isProxyChallenge = [[challenge protectionSpace] isProxy];
```
If a proxy error occurs, you can ask the user for the proxy username/password
and call the next fetcher's setProxyCredential: to provide the credentials for the
next attempt to fetch.

### Automatic Retry ###

The fetcher can optionally create a timer and reattempt certain kinds of
fetch failures (status codes 408, request timeout; 503, service unavailable;
504, gateway timeout; networking error NSURLErrorTimedOut.)  The user may set a retry selector to
customize the errors which will be retried.

Retries are done in an exponential-backoff fashion (that is, after 1 second,
2, 4, 8, and so on, with a bit of randomness added to the initial interval.)

Enabling automatic retries looks like this:
```
 myFetcher.retryEnabled = YES;
```
With retries enabled, the success or failure callbacks are called only
when no more retries will be attempted. Calling the fetcher's stopFetching
method will terminate the retry timer, without the finished or failure
selectors being invoked.

Optionally, the client may set the maximum retry interval:
```
 myFetcher.maxRetryInterval = 60.0; // in seconds; default is 60 seconds
                                    // for downloads, 600 for uploads
```
Also optionally, the client may provide a callback selector to determine
if a status code or other error should be retried.
```
 myFetcher.retrySelector = @selector(myFetcher:willRetry:forError:);
```

If set, the retry selector should have the signature:
```
  -(BOOL)fetcher:(GTMHTTPFetcher *)fetcher willRetry:(BOOL)suggestedWillRetry forError:(NSError *)error
```
and return YES to set the retry timer or NO to fail without additional
fetch attempts.

The retry method may return the `suggestedWillRetry` argument to get the
default retry behavior.  Server status codes are present in the
error argument, and have the domain kGTMHTTPFetcherStatusDomain. The
user's method may look something like this:
```
-(BOOL)myFetcher:(GTMHTTPFetcher *)fetcher willRetry:(BOOL)suggestedWillRetry forError:(NSError *)error {

    // perhaps examine [error domain] and [error code], or [fetcher retryCount]
    //
    // return YES to start the retry timer, NO to proceed to the failure
    // callback, or suggestedWillRetry to get default behavior for the
    // current error domain and code values.
    return suggestedWillRetry;
}
```

### Avoiding Duplicate Responses ###

The fetcher can track ETag headers from responses for each requested URL and
provide an "If-None-Match" header. This allows the server to save
bandwidth by providing a status message instead of repeated response
data.

To get this behavior, create the fetcher from an GTMHTTPFetcherService object,
and in the fetch callback look for an error with code 304
(kGTMHTTPFetcherStatusNotModified) like this:
```
- (void)myFetcher:(GTMHTTPFetcher *)fetcher finishedWithData:(NSData *)data error:(NSError *)error {
   if ([error code] == kGTMHTTPFetcherStatusNotModified) {
     // data is empty; use the data from the previous finishedWithData: for this URL
   } else {
     // handle other server status code
   }
}
```

The fetcher service object can optionally cache the ETagged responses in memory as well, so later fetchers will return the response data from the cache instead of the 304 status.
```
myFetcherService.shouldCacheETaggedData = YES;
```

The default cache size is 1 MB on iOS, 15 MB on Mac OS X. It can be set in the fetch history object:
```
myFetcherService.fetchHistory.memoryCapacity = 1000000;
```

### Uploading Multipart MIME Documents ###

The library includes a class for generating multipart MIME documents. The fetcher's `postStream` property may be used to specify the generated input stream to upload.

See the header `GTMMIMEDocument.h` for instructions on generating an NSInputStream with the MIME data. There is [an example page](MultipartPost.md) with code showing how to use the fetcher to post a multipart MIME form document.

## Fetcher Service Objects ##

Applications issuing a series of fetchers will typically want the fetchers to share a history, for limiting server load, managing cookies set by the server and avoiding fetching duplicate data.

Fetchers that do not need shared history may be generated independently, as shown above in "Fetching a Request".

Alternatively, fetchers created by a service class instance share a "memory" across a sequence of fetcher objects generated by the service.

A sequence of fetchers can be generated by a common instance of GTMHTTPFetcherService, like
```
GTMHTTPFetcherService *myService = [[GTMHTTPFetcherService alloc] init];
GTMHTTPFetcher* myFirstFetcher = [myService fetcherWithRequest:request1];
GTMHTTPFetcher* mySecondFetcher = [myService fetcherWithRequest:request2];
```

### Per-Host Throttling ###

The fetcher service limits the number of running fetchers making requests from the same host. If more than the limit is reached, additional requests to that host are deferred until existing fetches complete. By default, the number of running fetchers is 10, but it may be adjusted by setting the fetcher service's `maxRunningFetchersPerHost` property. Setting the property to zero allows an unlimited number of running fetchers per host.

### Shared Cookies ###

The fetcher service tracks cookies, allowing a sequence of fetchers to share a cookie jar, separate from the WebKit cookie storage.

### ETags and Response Caching ###

For fetch responses with ETag headers, the fetcher service can optionally remember the response headers. Future fetcher requests to the same URL will be given an "If-None-Match" header, telling the server to return a 304 Not Modified status if the response is unchanged, reducing the server load and network traffic. This behavior is controlled by the service's `shouldRememberETags` property, which defaults to NO.

Also optionally, the fetcher service can cache in memory the data that was returned in the responses that contained ETag headers. If a later fetch results in a 304 status, indicating the requested resource has not changed, the fetcher will return the cached data to the client along with a 200 status, hiding the 304. Caching is controlled by the `shouldCacheETaggedData` property, which defaults to NO.

See the header [GTMHTTPFetcherService.h](http://code.google.com/p/gtm-http-fetcher/source/browse/trunk/Source/GTMHTTPFetcherService.h) for the service properties.

### Stopping Fetchers ###

All fetchers issues by a fetcher service instance may be stopped immediately with the service object's `-stopAllFetchers` method.

## HTTP Logging ##

All traffic using GTMHTTPFetcher can be easily logged.  Call
```
  [GTMHTTPFetcher setLoggingEnabled:YES];
```
to begin generating log files. The project building the fetcher class must also include `GTMHTTPFetcherLogging.h` and `.m`.

### Finding the Log Files ###

For Mac OS X applications, log files are put into a folder on the desktop called `GTMHTTPDebugLogs`.

In the iOS simulator, the default logs location is the project derived data
directory in ~/Library/Application Support.  On the device, the
default logs location is the application's documents directory on the device.

**Tip**: Search the Xcode console for the message **GTMHTTPFetcher logging to** followed by the path of the logs folder. Copy the path, including the quotation marks, and open it from the Mac’s terminal window, like
```
  open "/Users/username/Library/Application Support/iPhone Simulator/6.1/Applications/C2D1B09D-2BF7-4FC9-B4F6-BDB617DE76D5/GTMHTTPDebugLogs"
```
**Tip**: Use the Finder's "Sort By Date" to find the most recent logs.

Each run of an application generates a separate log.  For each run, an html
file is created to simplify browsing the run's http requests and responses.

To view the most recently saved logs, use a web browser to open the symlink named MyAppName\_log\_newest.html (for whatever your application's name is) in the logging directory.

**Tip**: Set the fetcher's `comment` property or call `setCommentWithFormat:` before beginning a fetch to assign a short description for the request's log entry.

Each request's log is also available via the fetcher `log` property when the fetcher completes; this makes viewing a fetcher's log convenient during debugging.

**iOS Tip**: The library includes an iOS view controller, `GTMHTTPFetcherLogViewController`, for browsing the http logs on the device.

**iOS Note**: Logging support is stripped out in non-DEBUG builds by default. This can be overridden by defining DEBUG=1 for the project, or by explicitly defining STRIP\_GTM\_FETCH\_LOGGING=0 for the project.

# Questions and Comments #

Additional documentation is available in the header files.

If you have any questions or comments about the library or this documentation, please join the [discussion group](http://groups.google.com/group/google-toolbox-for-mac).