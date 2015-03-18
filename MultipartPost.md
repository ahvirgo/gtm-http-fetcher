# Example of uploading a multipart body #

```
#import "GTMHTTPFetcher.h"
#import "GTMMIMEDocument.h"

- (void)postMultipartBody {
  GTMHTTPFetcher *fetcher = [GTMHTTPFetcher fetcherWithURLString:@"http://www.example.net"];

  // To upload a multipart MIME body, create the document, then add
  // parts (headers and data)
  GTMMIMEDocument *doc = [GTMMIMEDocument MIMEDocument];

  // The first part will be a simple string
  NSDictionary *headers = [NSDictionary dictionaryWithObject:@"text/plain"
                                                      forKey:@"Content-Type"];
  NSString *str = @"Apparently there is nothing that cannot happen today";
  NSData *body = [str dataUsingEncoding:NSUTF8StringEncoding];
  [doc addPartWithHeaders:headers
                     body:body];

  // The second part will be an image file
  NSString *imageFilePath = @"/myphoto.jpg";

  NSString *mimeType = @"image/jpeg"; // We will fall back on image/jpeg as a default
  
  // Example of finding a file MIME type dynamically from its file extension
  //
  // NSString *imageExtn = [imageFilePath pathExtension];
  // CFStringRef fileUTI = UTTypeCreatePreferredIdentifierForTag(kUTTagClassFilenameExtension, (CFStringRef)imageExtn, NULL);
  // if (fileUTI) {
  //   CFStringRef mimeTypeCF = UTTypeCopyPreferredTagWithClass (fileUTI, kUTTagClassMIMEType);
  //   if (mimeTypeCF) {
  //     mimeType = [NSString stringWithString:(NSString *)mimeTypeCF];
  //     CFRelease(mimeTypeCF);
  //   }
  //   CFRelease(fileUTI);
  // }
  
  headers = [NSDictionary dictionaryWithObjectsAndKeys:
             mimeType, @"Content-Type",
             [imageFilePath lastPathComponent], @"Slug",
             nil];
  body = [NSData dataWithContentsOfFile:imageFilePath];
  [doc addPartWithHeaders:headers
                     body:body];

  // Now create a stream from the document, and make the stream the
  // fetcher POST body
  NSInputStream *stream = nil;
  [doc generateInputStream:&stream
                    length:NULL
                  boundary:NULL];
  if (stream) {
    [fetcher setPostStream:stream];
    [fetcher beginFetchWithCompletionHandler:^(NSData *data, NSError *error) {
      // Callback
      NSLog(@"data:%@  error:%@", data, error);
    }];
  }
}
```