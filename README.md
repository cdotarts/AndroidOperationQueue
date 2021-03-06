[![Android Arsenal](https://img.shields.io/badge/Android%20Arsenal-AndroidOperationQueue-green.svg?style=true)](https://android-arsenal.com/details/1/3552)

# AndroidOperationQueue
AndroidOperationQueue is tiny serial operation queue for Android Development. 

## Setup Gradle

```groovy
dependencies {
	...
	compile 'kr.pe.burt.android.lib:androidoperationqueue:0.0.2'
}
```

## Examples

You can see the examples using AndroidOperationQueue at [https://github.com/skyfe79/AndroidOperationQueue/tree/master/examples](https://github.com/skyfe79/AndroidOperationQueue/tree/master/examples) 

 * [Simple example](https://github.com/skyfe79/AndroidOperationQueue/tree/master/examples/DownloadImage)
 * [Image Download with cache](https://github.com/skyfe79/AndroidOperationQueue/tree/master/examples/ImageDownloadWithCache)
 * [Image Download without cache](https://github.com/skyfe79/AndroidOperationQueue/tree/master/examples/MultipleImageDownload) 

## Image Download without cache
![](examples/art/image_download_without_cache.gif)

## Image Download with cache
![](examples/art/image_download_with_cache.gif)

## APIs

### Add operations

You can add operation by using below methods.

 * addOperation()
 * addOperationAtFirst()
 * addOperationAtTime()
 * addOperationAfterDelay()

```java
AndroidOperationQueue queue = new AndroidOperationQueue("JobQueue");
queue.addOperation(new Operation() {
	@Override
	public void run(AndroidOperationQueue q, Bundle bundle) {
		
		// doing job #1
	
	}
});	

queue.addOperation(new Operation() {
	@Override
	public void run(AndroidOperationQueue q, Bundle bundle) {
		
		// doing job #2
	
	}
});	

```

### Remove operations

You can remove operation by using below methods.

 * removeOperation()
 * removeOperations()
 * removeAllOperations()

```java
Operation operation = new Operation() {
	@Override
	public void run(AndroidOperationQueue q, Bundle bundle) {
		
		// doing job #n
	
	}
};

queue.addOperation(operation);

...

queue.removeOperation(operation);

```

### Start & Stop OperationQueue

You can start or stop Android Operation Queue by using below methods.

 * start()
 * stop()

`stop()` method removes all pending operations that are in operation queue. 

```java
AndroidOperationQueue queue = new AndroidOperationQueue("JobQueue");

// add multiple operations
... 

// start queue
queue.start();


// stop queue
queue.stop();


// you can add operation to same queue.
queue.addOperation( ... );

queue.start();
```

### Share common data among operations via Bundle.

You can share data by using Bundle which is in AndroidOperationQueue. If you want to make operation chain, you want to send some result from the current operation to the next operation. You can add up 1, 2 and 3 by using operation and bundle like below. 

```java
queue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue q, Bundle bundle) {
        bundle.putInt("sum", 1);
    }
});
queue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue q, Bundle bundle) {
        int sum = bundle.getInt("sum");
        bundle.putInt("sum", sum + 2);
    }
});
queue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue q, Bundle bundle) {
        int sum = bundle.getInt("sum");
        bundle.putInt("sum", sum + 3);
    }
});
queue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue q, Bundle bundle) {
        int sum = bundle.getInt("sum");
        Log.v("SUM", String.format("1+2+3 = %d", sum));
    }
});
```

Output is

```
V/SUM: 1+2+3 = 6
```

### Example for downloading images with cache

```java
AndroidOperationQueue downloadQueue = new AndroidOperationQueue("DownloadQueue");

downloadQueue.stop();

downloadQueue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue queue, Bundle bundle) {
        String url = item.getImageURL();
        bundle.putString("url", url);
        AndroidOperation.runOnUiThread(new Runnable() {
            @Override
            public void run() {
                holder.image.setImageBitmap(null);
                holder.line.setVisibility(View.INVISIBLE);
                holder.progressBar.setVisibility(View.VISIBLE);
            }
        });
    }
});

downloadQueue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue queue, Bundle bundle) {
        String url = bundle.getString("url");
        if(url == null) {
            queue.stop();
        }

        // check the url if there is the url in the memory cache
        if(Cache.sharedInstance().hasURLInMemoryCache(url) == true) {
            final Bitmap bitmap = Cache.sharedInstance().getBitmapFromMemoryCache(url);
            if(bitmap != null) {
                AndroidOperation.runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        holder.image.setImageBitmap(bitmap);
                        holder.line.setVisibility(View.VISIBLE);
                        holder.progressBar.setVisibility(View.INVISIBLE);
                    }
                });
                queue.stop();
            }
        }
    }
});

downloadQueue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue queue, Bundle bundle) {

        String url = bundle.getString("url");
        if(url == null) {
            queue.stop();
        }

        //check the url if there is the url in the file cache
        if(Cache.sharedInstance().hasURLInFileCache(url) == true) {
            final String path = Cache.sharedInstance().getFilePathFromFileCache(url);
            final Bitmap bitmap = BitmapFactory.decodeFile(path);

            if(bitmap != null) {
                Cache.sharedInstance().putBitmapInMemoryCache(url, bitmap);
                AndroidOperation.runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        holder.image.setImageBitmap(bitmap);
                        holder.line.setVisibility(View.VISIBLE);
                        holder.progressBar.setVisibility(View.INVISIBLE);

                    }
                });
                queue.stop();
            }
        }
    }
});

downloadQueue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue queue, Bundle bundle) {

        String url = bundle.getString("url");
        if(url == null) {
            queue.stop();
        }


        // there is no bitmap on memory or file then download bitmap from url.
        final Bitmap bitmap = downloadBitmapFromURL(url);
        if(bitmap != null) {
            Cache.sharedInstance().putBitmapInMemoryCache(url, bitmap);
            AndroidOperation.runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    holder.image.setImageBitmap(bitmap);
                    holder.line.setVisibility(View.VISIBLE);
                    holder.progressBar.setVisibility(View.INVISIBLE);
                }
            });

            String path = FileUtils.generateTempFileAtExternalStorage("ImageDownloadWithCache", "temp_", ".jpeg");
            boolean success = saveBitmapToPath(bitmap, path);
            if(success) {
                Cache.sharedInstance().putPathInFileCache(url, path);
            }
        }
    }
});

downloadQueue.start();
```

### Operation Utils

#### runOnUiThread

If you want to update ui element after some background work, you should do it on main thread(ui thread). AndroidOperation provide convenient class method like below.

```java
queue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue q, Bundle bundle) {
    
    	 // doing background work
    	
        AndroidOperation.runOnUiThread(new Runnable() {
            @Override
            public void run() {
                textView.setText("It's completed");
            }
        });
    }
});
```

#### runOnUiThreadAfterDelay

You can also use main thread after some time like below.

```java
queue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue q, Bundle bundle) {
    
    	 // doing background work
    	
        AndroidOperation.runOnUiThreadAfterDelay(new Runnable() {
            @Override
            public void run() {
                textView.setText("It's completed");
            }
        }, 1000);
    }
});
```

#### sleep

You can sleep current opertation thread for some time like below.

```java
queue.addOperation(new Operation() {
    @Override
    public void run(AndroidOperationQueue q, Bundle bundle) {
    
    	 // doing background work
    	
        AndroidOperation.runOnUiThreadAfterDelay(new Runnable() {
            @Override
            public void run() {
                textView.setText("It's completed");
            }
        }, 1000);
        
        AndroidOperation.sleep(1000); // sleep for 1 second.
    }
});
```

## MIT License

The MIT License (MIT)

Copyright (c) 2016 Sungcheol Kim, [https://github.com/skyfe79/AndroidOperationQueue](https://github.com/skyfe79/AndroidOperationQueue)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.