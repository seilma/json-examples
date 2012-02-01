The C# example code is located in a [Git repository here]().

This code requires the following dependencies:

> [fastJSON](http://www.codeproject.com/Articles/159450/fastJSON)- v1.9.6
>
> [JSON.NET](http://james.newtonking.com/pages/json-net.aspx)- v4.0

**Installation**

This depends on the project files being at:
- fastJSON: '`./fastJSON\\ 1.9.6/fastJSON/`'
- JSON.Net: '`./Json40r6/Source/Src/Newtonsoft.Json`'

To install, compile using Visual Studio, Mono Develop, or xbuild:

    xbuild example.csproj

There are examples using both libraries, so only one is necessary.

This example will:

- get the `/geolocation.json` resource from a Spotter
- grab the '`altitude`' value and do some simple modifications to it
    - example using fastJSON
    - example using JSON.NET
- post the changed JSON data back to the Spotter

**If using fastJSON**

The built-in `toJSON` function adds a wrapper to the output, which causes the example to fail. To fix this, remove lines 47 and 59 from `fastJSON/JsonSerializer.js`. These line numbers are from v1.9.6, and look something like this:

    sb.Append("{\"$types\":{")
    sb.Append("},");

This fix has been proposed on the project home page, so it may be fixed in later versions. A more robust fix would be to expose the internal `_useGlobalTypes` variable or alter the logic. The more robust fix is recommended if this is to be used in production, but for this example, removing those two lines will suffice.

### Code Walkthrough ###

There are four major functions:

- `GetSettings` - example of making an HTTP GET request to get data
- `SetSettings` - example of making an HTTP POST request to set data
- `ParseSettingsFastJSON` - example of using the fastJSON library
- `ParseSettingsJSONNet` - example of using the JSON.NET library

Both HTTP functions use built-in libraries.

#### GetSettings ####

Takes a string representing a URL without a path.

    // create the request object
    HttpWebRequest req = (HttpWebRequest)WebRequest.Create(url + "?minWait=1000");

    // get a response
    HttpWebResponse res = (HttpWebResponse)req.GetResponse();

This sets up and makes the request. Notice that we were able to put query parameters directly into the URL. Since these are just settings files, the query parameters don't make much sense, but this is how to append them.

The call to `req.GetResponse()` blocks until the server responds. Once the response is received, we grab the stream and all of the data in it:

    // get a stream to the response body
    Stream resStream = res.GetResponseStream();

    // set up the variables we'll need
    int count = 0;
    byte[] buf = new byte[8192];
    StringBuilder sb = new StringBuilder();
    while ((count = resStream.Read(buf, 0, buf.Length)) > 0) {
        sb.Append(Encoding.UTF8.GetString(buf, 0, count));
    }

Once all of the data is received, we just pass this back.

#### SetSettings ####

Sending the settings up to the server is quite similar, but we'll need to set more things on the request. In this example, `settings` is the result from `GetSettings`.

    HttpWebRequest req = (HttpWebRequest)WebRequest.Create(url + RESOURCE + "/settings");
    req.Method = "POST";
    req.ContentType = "application/json";
    req.ContentLength = settings.Length;

Here, we specify that this is a POST request, whose content is JSON encoded. This is very important, because the Spotters only respond to data that is specifically stated as being JSON. We go ahead and set the content length because we know it.

Next, we write the data to the request stream:

    // get the request stream and write our data back to it
    Stream writeStream = req.GetRequestStream();
    writeStream.Write(bytes, 0, bytes.Length);

And finally get the response:

    HttpWebResponse res = (HttpWebResponse)req.GetResponse();

Once we have the response, we grab the data again as before:

    Stream resStream = res.GetResponseStream();

    StringBuilder sb = new StringBuilder();
    byte[] buf = new byte[8192];
    int count = 0;
    while ((count = resStream.Read(buf, 0, buf.Length)) > 0) {
        sb.Append(Encoding.UTF8.GetString(buf, 0, count));
    }

The response tells us if the request succeeded, or if it failed.

#### ParseSettingsFastJSON ####

In this example, we'll make a lot of assumptions about the structure of the data. In practice, more care should be taken in ensuring that the data is well formed and as we expect.

Objects in `fastJSON` are represented as a `Dictionary<string, object>`. To start off, we'll need to parse our object:

    Dictionary<string, object> jsonData = fastJSON.JSON.Instance.Parse(settings) as Dictionary<string, object>;

From here on out, it's just like working with a regular Dictionary. To start, grab the `result` object that has all of our settings:

    Dictionary<string, object> result = jsonData["result"] as Dictionary<string, object>;

Grab the altitude and manipulate it:

    float alt = float.Parse(result["altitude"] as string);
    if (alt > 2000) {
        alt = 747;
    } else {
        alt += 1000;
    }

Then put it back in:

    result["altitude"] = alt;

Then convert it back to a string:

    JSON.Instance.ToJSON(result);

This can now be sent back to the server! Notice that we're only sending back the result part of it. The header should be discarded.

#### ParseSettingsJSONNet ####

In this example, we'll make a lot of assumptions about the structure of the data. In practice, more care should be taken in ensuring that the data is well formed and as we expect.

To start off, we'll parse our settings into an object:

    JObject obj = JObject.Parse(settings);

Then grab the result object inside:

    JToken result = obj["result"];

Now that we have our result object, let's grab the altitude and manipulate it:

    float alt = result["altitude"].Value<float>();
    if (alt > 2000) {
        alt = 747;
    } else {
        alt += 1000;
    }

Now that we're done, we'll put it back in:

    result["altitude"] = alt;

Turning this back into a string is as simple as calling it's ToString() method:

    result.ToString();

That's it! It can now be sent to the server. Notice that we're only sending back the result part of it--the header should be discarded.
