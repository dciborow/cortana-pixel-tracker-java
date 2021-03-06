#Cortana Intelligence Marketing
# Pixel Tracker Java Service

## Pixel API
### Pixel.java
An interface and an abstract class are provided to base a Pixel API upon. The interface, **Pixel.java** first contains the request mapping for the path **/pixel**, using the Spring **@RequestMapping** annotation. It next sets the produced media type of the API to GIF. Lastly, it specifically requires GET requests, and will aceppt all headers. While these can be overridden, building future APIs upon this interface will help maintain backward comptabiltiy. The **@RequestParam** annotation is used to pass the query string from the request to the java class. 

```java
package com.microsoft.azure.server.pixeltracker.api;

import ...;

public interface Pixel {

    @RequestMapping(value = "/pixel",
            produces = MediaType.IMAGE_GIF_VALUE,
            method = RequestMethod.GET, headers = "Accept=*/*")
    byte[] pixel(@RequestParam Map<String, String> queryParameters);
}```

The abstract implementation, **AbstractPixel.java**, adds some basic functionality to the class. 
package com.microsoft.azure.server.pixeltracker.api;

### AbstractPixel.java
```java
package com.microsoft.azure.server.pixeltracker.api;
import ...;

public class AbstractPixel implements Pixel {
    private static Logger logger = LogManager.getLogger();
    private final Handler handlers;

    public AbstractPixel(Handler handlers) throws Exception {...}

    @Override
    public byte[] pixel(Map<String, String> queryParameters) {...}
}
```

First, it creates the base constructor, which performs a fast fail check on the handlers input.

```java
    public AbstractPixel(Handler handlers) throws Exception {
        if (handlers == null) throw new Exception("Handler not expected to be null. Check your Spring Config!");
        this.handlers = handlers;
    }
```    
 Second, it declares the functionality of the pixel operation that was stubbed in the interface. This establishings a default logic for the API that could be overridden if desired. Log4j2 is used to record any unexpected errors. 

```java
    @Override
    public byte[] pixel(Map<String, String> queryParameters) {
        try {
            handlers.handle(new RequestImpl(queryParameters));
        } catch (Exception e) {
            logger.error(e);
        }
        return new byte[]{0};
    }
```

##Handlers
###Handler.java
**Handler.java** is an interface for chaining together transformations. Each handler will implement a *handle*, *strategy* and *setNextOperation* method.

```java
package com.microsoft.azure.server.pixeltracker.api.handlers;

import ...;

public interface Handler {
    Future<Boolean> handle(Request request) throws Exception;

    void strategy(Request request) throws Exception;

    Handler setNextOperation(Handler childHandler);    
}
```

###AbstractHandlerjava
**AbstractHandler.java** is an abstract class that makes implemnting the interface simpler. 

```java
package com.microsoft.azure.server.pixeltracker.api.handlers.impl;

import ...;

public abstract class AbstractHandler implements Handler {
    private Handler childHandler = null;

    @Override
    public Handler setNextOperation(Handler childHandler) {...}

    @Override
    public Future<Boolean> handle(Request request) throws Exception {...}

    private Handler setChildHandler(Handler childHandler) {...}

    public Future<Boolean> chainHandler(Request request) throws Exception {...}
}
```

*Handle* will be called on each API request. The abstract class implments a common flow of *strategy*, then *chainHandler* that should be followed on each request. *Strategy* is not implemented in the abstract class, and must be written for each implementation. 

```java
    @Override
    public Future<Boolean> handle(Request request) throws Exception {
        strategy(request);
        return chainHandler(request);
    }
```
*chainHandler* determines the order in which transformations are performed. The default implementation is to synchronously call the next handler. This default behavior can be overridden if desired.

```java
    public Future<Boolean> chainHandler(Request request) throws Exception {
        return childHandler == null
                ? new AsyncResult<>(request.isSuccess())
                : this.childHandler.handle(request);
    }
```

###JsonQueryStringHandler.java
The **JsonQueryStringHandler.java** handler processes the query string from the web request and converts it into a JSON. 

```java
package com.microsoft.azure.server.pixeltracker.api.handlers.impl;

import ...;

public class JsonQueryStringHandler extends AbstractHandler {
    private static Logger logger = LogManager.getLogger();
    private final String PARAMDELIM = "&";

    @Override
    public void strategy(Request request) throws Exception {...}

    private boolean isNotEmpty(String queryString) {...}

    private Stream<String> getParamsFrom(String queryString) {...}

    private void putIntoOutput(String param, Request request) {...}
}
```

The *strategy* first retrieves the query string from the request object. It then checks to see if the there were entries in the query string. Next, it places each parameter into the output json, stored in the request object. 

```java
   @Override
    public void strategy(Request request) throws Exception {
        logger.trace("Json Query String Strategy");
        String queryString = request.getQueryString();
        if (isNotEmpty(queryString))
            getParamsFrom(queryString)
                    .forEach(param -> putIntoOutput(param, request));
        else request.setSuccess(false);
    }
```

###EventHubSendHandler.java

The **EventHubSendHandler.java** takes the JSON stored in request, and sends it to Event Hub for storage and processing. Spring is used to wire the Event Hub configurations. The configurations will be declared as beans in the main class of our service.

```java
/*
 * Copyright (c) Microsoft. All rights reserved.
 * Licensed under the MIT license. See LICENSE file in the project root for full license information.
 */
package com.microsoft.azure.server.pixeltracker.api.handlers.impl;

import com.microsoft.azure.eventhubs.EventData;
import com.microsoft.azure.eventhubs.spring.EventHubAutoConfiguration;
import com.microsoft.azure.server.pixeltracker.api.model.Request;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;

import java.io.UnsupportedEncodingException;
import java.util.concurrent.CompletableFuture;

public class EventHubSendHandler extends AbstractHandler {
    private static Logger logger = LogManager.getLogger();

    @Autowired
    private EventHubAutoConfiguration ehClient;
    private final String ENCODING = "UTF-8";

    public EventHubSendHandler(EventHubAutoConfiguration ehClient) {...}

    @Override
    public void strategy(Request request) throws Exception {...}

    private CompletableFuture<Void> sendWithEventHubClient(Request request) throws UnsupportedEncodingException {...}

    private EventData eventDataFrom(Request request) throws UnsupportedEncodingException {...}
}
```

The constructor takes an EventHubAutoConfiguration object as an input. This is used to access a singleton event hub client. 

```java
    public EventHubSendHandler(EventHubAutoConfiguration ehClient) {
        assert ehClient != null;
        this.ehClient = ehClient;
    }
```

The **strategy** checks to see that the previous handlers were succesful, and then sends the JSON to event hub. 
```java
    @Override
    public void strategy(Request request) throws Exception {
        logger.trace("Event hub Send Strategy");
        if (request.isSuccess()) sendWithEventHubClient(request);
    }
```