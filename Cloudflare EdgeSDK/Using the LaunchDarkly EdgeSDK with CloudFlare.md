# Using the LaunchDarkly EdgeSDK with CloudFlare

## Overview

This guide explains how to connect LaunchDarkly with Cloudflare Workers using [LaunchDarkly's Cloudflare Edge SDK](https://docs.launchdarkly.com/sdk/server-side/node-js/cloudflare-edge-sdk).

Cloudflare Workers are serverless functions that run "at the edge" within Cloudflare's Content Delivery Network (CDN). Unlike traditional serverless functions that are deployed to a single region, edge functions like Cloudflare Workers are deployed across a global CDN network. This means that users' requests are routed to the nearest CDN, allowing the function response to be extremely fast.

LaunchDarkly provides a Cloudflare Workers integration that synchronizes flags and flag values with Cloudflare to make accessing flag values from within a Worker available without any processing delay.

This guide will walk you through the steps of getting set up within Cloudflare as well as setting up the LaunchDarkly integration. It will also walk you through several examples of utilizing LaunchDarkly flags within a Worker to alter the response sent back to the user based upon flag values.

## Getting set up

For this guide, we've created a sample CloudFlare Workers project that demonstrates several uses of flags within a Worker. You can find the finished code [on GitHub](https://github.com/remotesynth/cfworkers-ld). You can use this code as reference as you work through steps in the guide. You can view the finished page running on Cloudflare Workers [here](https://cfworkers-ld.remotesynth.workers.dev/).

If you would like to follow along creating the project in this guide, you can begin by cloning the site assets [in this repository](https://github.com/remotesynth/cfworkers-ld-assets). The assets contain the code for the web site without the Worker file or the custom client-side JavaScript shared in the examples belwo. In order to compile the site, you'll need to [install Hugo](https://gohugo.io/getting-started/installing/), a static site generator built in Go. The easiest way to install Hugo is using Homebrew on Mac:

```bash
brew install hugo
```

Or via Chocolatey on Windows:

```bash
choco install hugo -confirm
```

### Setting up Cloudflare CLI

The quickest way to get set up with Cloudflare Workers locally is to use [Cloudflare's Worker CLI](https://developers.cloudflare.com/workers/cli-wrangler/install-update) called Wrangler via npm:

```bash
npm install -g @cloudflare/wrangler
```

Or via Yarn:

```bash
yarn global add @cloudflare/wrangler
```

Once it is installed, you'll need to use the `wrangler login` command, which will open a browser window allowing you to log into your Cloudflare account to authenticate the CLI. For additional CLI authentication methods, you can reference the [Wrangler docs](https://developers.cloudflare.com/workers/cli-wrangler/authentication).

### Setting up local development for Cloudflare Workers

This guide uses [Workers Sites](https://developers.cloudflare.com/workers/platform/sites) that allow you to deploy an application buit with a static site generator (SSG) or front end framework to a worker. You can follow the [instructions for starting from an existing project](https://developers.cloudflare.com/workers/platform/sites/start-from-existing) to publish a simple static site built with the [Hugo SSG](https://gohugo.io).

> The Wrangler 2 beta supports a new, streamlined Workers integration with Pages, Cloudflare's solution for deploying sites. However, it does not currently allow for providing a custom Webpack configuration, which is required for LaunchDarkly's Cloudflare Edge SDK.

Open the terminal in the folder containing the site assets from GitHub and enter the following command:

```bash
wrangler init --site my-static-site
```

This should have created a `wrangler.toml` and a `workers-site` directory within the project. Open the `wrangler.toml` and add the following line that tells the worker to find the site files in the `/public` folder, which is where Hugo outputs them by default:

```toml
site = { bucket = "./public" }
```

While you haven't actually created a Worker yet, we can run the site locally using Wrangler. The first step is to build the site using Hugo and then run a local server with Wrangler.

```bash
hugo
wrangler dev
```

You should be able to view the site at the URL and port indicated in the console, typically `localhost:8787`.

### A simple Cloudflare Worker

Wrangler will generate a Cloudflare worker for you. As noted, it will run as is, but it does contain a lot of extraneous code that you won't need for this guide. Feel free to replace the existing worker code in `/worker-site/index.js` with the following simplified code. This will make it easier to update using the examples in the remainder of this guide.

The Worker gets the existing page content from the Cloudflare's KV (key value store) where all the site's assets are cached and returns that in the response to the user's browser.

```javascript
import { getAssetFromKV } from "@cloudflare/kv-asset-handler";

addEventListener("fetch", (event) => {
  event.respondWith(handleEvent(event));
});

async function handleEvent(event) {
  let options = {};

  try {
    const page = await getAssetFromKV(event, options);
    const response = new Response(page.body, page);

    return response;
  } catch (e) {
    console.log(e);
    return new Response(e.message || e.toString(), { status: 500 });
  }
}
```

### Setting up the LaunchDarkly Cloudflare integration

LaunchDarkly's Cloudflare integration synchronizes flag data from a LaunchDarkly project and environment with a KV (key value store) connected to your Worker in Cloudflare. This means that the latest flag data is immediately available to the LaunchDarkly client within your Worker without the need for additional external calls. This makes it extremely fast.

In order to set up the integration, you'll need a minimum of one KV created to sync values with. To set this up, first make sure that your account ID is in the `wrangler.toml` that was created by the Wrangler CLI. Your account ID is listed on the overview page of the Cloudflare Workers dashboard or you can get it by using `wrangler whoami` from the command line.

```toml
account_id = "<YOUR_CLOUDFLARE_ACCOUNT_ID>"
```

Make note of your account ID as we'll need it to enable the Cloudflare integration.

Next, create a new KV namespace by entering the following command. Ensure that you run this command from within your project folder:

```bash
wrangler kv:namespace create "MY_KV"
```

The namespace name will be a combination of the namespace name you provided (`MY_KV`) and the project name. For example, if my project was named `cfworkers-ld`, the name of the created namespace will be `cfworkers-ld-MY_KV`. When the namespace has been created, Wrangler will return the namespace ID of the new KV namespace.

Open `wrangler.toml` and add the namespace ID to the `kv_namespaces` configuration. If this configuration key does not exist yet, create it. If it does exist with the KV namespace that was created for your site assets, add the new namespace to the array of namespaces.

```toml
kv_namespaces = [
  { binding = "MY_KV", id = "<NAMESPACE_ID>" },
  { binding = "SITE_ASSETS", preview_id = "43ecd3c7d05f45caa947fb48fd7b7c83", id = "a0a8f76d2fe14947826adc1453c2e90c" },
]
```

Make note of the namespace ID as we'll need it to enable LaunchDarkly's Cloudflare integration.

> If you plan to use the [Cloudflare Workers preview service](https://cloudflareworkers.com/), you will need to create a preview namespace as well. Follow the steps in the [Cloudflare integration docs](https://docs.launchdarkly.com/integrations/cloudflare#creating-a-cloudflare-worker-with-the-launchdarkly-cloudflare-edge-sdk) to set this up.

The last piece of information we'll need to enable Cloudflare integration is a Cloudflare API token. From the Workers overview page in the Cloudflare dashboard, under "Get started" on the right hand navigation links, click the [API tokens](https://dash.cloudflare.com/profile/api-tokens) link. Then press the "Create Token" button.

Next, click the "Get started" button next to the "Create custom token" option.

![Custom token option](CF-SDK-custom-token.png)

Give the token a name (for example, "LaunchDarkly") and then under "Permissions" choose the following options from the dropdowns:

1. Account
2. Workers KV Storage
3. Edit

![Cloudflare token permissions](CF-SDK-token-creation.png)

Click the "Continue to Summary" button and then "Create Token". Copy the token from the subsequent page and make note of it will not be shown again.

> Detailed instructions on creating a token can be found in the [Cloudflare docs](https://developers.cloudflare.com/api/tokens/create).

Next you need to set up the integration within the LaunchDarkly dashboard. Go to the [Integrations page](https://app.launchdarkly.com/default/integrations/) and search for Cloudflare. Click the "Add integration" button, which will bring up a form requesting the following details:

* **Name** – This is optional but is useful for determining which Worker namespace this is connected to when you have multiple connections.
* **Environment** – Which LaunchDarkly environment will be used when syncing flags and values with the KV on Cloudflare.
* **Account ID** – Your Cloudflare account ID.
* **KV Namespace ID** – The namespace ID for the KV connected to your Worker. Note that if you also created a preview KV, you'll need a separate integration set up using the preview KV namespace ID as well.
* **API token** – The Cloudflare API token you just created.

![LaunchDarkly Cloudflare integration set up](CF-SDK-integration.png)

Click the "Save configuration" button. If you want to verify that the information is correct, click the "Validate Connection" button. If everything connected properly, you're ready to begin adding LaunchDarkly into your Worker.

### Initializing LaunchDarkly within a Worker

Before you can get flag values from within a Worker, you'll need to import and intialize the Cloudflare Edge SDK. Begin by installing the SDK.

```bash
npm install launchdarkly-cloudflare-edge-sdk
```

Open or create an `index.js` file within the `workers-site` folder of your project. This folder shoud have been created by the `wrangler init` command you ran earlier. At the top of the file, add the `require` statement to import the SDK into the Worker file. In addition, you'll need to initialize the variable that will contain the instance of the LaunchDarkly client when it is initialized.

```javascript
const { init } = require("launchdarkly-cloudflare-edge-sdk");
let ldClient;
```

Within your Worker, there should be a event listener for the `fetch` event that calls a `handleEvent()` function. The `fetch` event is triggered by any incoming HTTP request. You can initialize the LaunchDarkly client within this function. You'll pass it the KV namespace defined within your `wrangler.toml` and your LaunchDarkly client ID, which can be found in your [account settings](https://app.launchdarkly.com/settings/projects).

```javascript
if (!ldClient) {
  ldClient = init(MY_KV, "<LAUNCHDARKLY_CLIENT_ID>");
  await ldClient.waitForInitialization();
}
```

Now you are ready to use get flag variations within your application.

### Cloudflare's HTMLRewriter class

The examples below make use of a powerful feature that Cloudflare Workers provides called [HTMLRewriter](https://developers.cloudflare.com/workers/runtime-apis/html-rewriter). HTMLRewriter is a JavaScript class that you can leverage within Cloudflare worker code to modify the content of the response being sent back to the user. This allows you to do things like modify the page's HTML or change text in the response. To better understand the code in the examples that follow, let's cover some of the basics of the HTMLRewriter. 

A new instance of the HTMLRewriter class can be constructed as follows:

```javascript
const rewriter = new HTMLRewriter();
```

An instance of HTMLRewriter provides two functions:

1. `on()` listens for any selected elements on the page. Elements are selected via [selectors](https://developers.cloudflare.com/workers/runtime-apis/html-rewriter#selectors) that offer a subset of standard [CSS selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors) commonly used for selecting elements with the document object model (DOM). Each matching element is passed to the element handler that you define.
2. `onDocument()` responds to the entire HTML document, passing the contents of that document to a document handler that you specify.

Corresponding to the above, there are two types of handlers:

1. [Element Handlers](https://developers.cloudflare.com/workers/runtime-apis/html-rewriter#element-handlers) specify the code that will run on each matching element returned by the selector specified in `on()`. This can be used to add, update or remove matching elements and content from within the HTML response.
2. [Document Handlers](https://developers.cloudflare.com/workers/runtime-apis/html-rewriter#document-handlers) specify the code that runs when the entire HTML document is received. This can be used to modify the doctype, modify the text or add code that runs at the end of the document.

Two of the below examples will make use of element handlers to modify the HTML response with a Cloudflare Worker before it is ever received by the end user. If you'd like to know more about HTMLRewriter, check the [Cloudflare docs](https://developers.cloudflare.com/workers/runtime-apis/html-rewriter).

## Bootstrapping client-side flag values

A persistent problem with modifying the client UI on the web using JavaScript is the delay between when a UI element is initially rendered and when the update runs in the script. This causes what can be called a "flash of initial content", where the initial rendering flashes on screen before it gets updated. A common example of this is login/sign up links briefly rendering before getting updated with the logged in user's information.

Imagine a scenario using a LaunchDarkly flag to enable or disable a feature within the browser UI. You definitely do not want the feature to display, however briefly, before disappearing. This could cause confusion and possibly frustration on the part of the user. While LaunchDarkly's client SDKs provide tools caching in LocalStorage to minimize these types of issues, the nature of how JavaScript runs in the browser means that any fully client-side solution cannot completely eliminate the delay, though there are methods to obscure it. Cloudflare Workers will allow us to actually eliminate that delay by directly injecting our client-side flag values into the HTML before the request is ever received by the browser.

Within your Cloudflare Worker file, you'll need to first instantiate an instance of the HTMLRewriter class. This can be placed prior to the `addEventListener()` block within `/workers-site/index.js`.

```javascript
const rewriter = new HTMLRewriter();
```

You'll want to inject these values in the HTML `<head>` so that they are available immediately before any of the DOM or JavaScript gets processed by the browser engine. You can do this by having the HTMLRewriter listen for the head element. Place this directly after the `rewriter` instantiation.

```javascript
rewriter.on("head", new FlagsStateInjector());
```

When a `<head>` element is found, it will create a new instance of the `FlagStateInjector` class. This class contains an element handler that injects the flag values into a `<script>` element within the `<head>`. In this instance, the LaunchDarkly client is only pulling flag values that have client-side SDK availability enabled using an anonymous user. If you have user details available within your Worker, you could pass them here instead of the anonymous user. This code can be placed anywhere within the Worker file that is outside of existing function blocks. For instance, at the top of the file immediately after the `ldClient` variable instantiation.

```javascript
class FlagsStateInjector {
  async element(element) {
    // fetch all flag values for client-side SDKs as evaluated for an anonymous user
    // use a more appropriate user key if needed
    const user = { key: "anonymous" };
    const allFlags = await ldClient.allFlagsState(user, {
      clientSideOnly: true,
    });
    element.append(
      `<script>window.ldFlags = ${JSON.stringify(allFlags)}</script>`,
      { html: true }
    );
  }
}
```

The last small change you need to make to your Worker is to call the HTMLRewriter instance to alter the response. Instead of returning just `response` from within the `handleEvent()` method, you'll wrap it in the rewriter's `transform()` method.

```javascript
return rewriter.transform(response);
```

Finally, you need tell LaunchDarkly to [bootstrap the client](https://docs.launchdarkly.com/guides/platform-specific/static-sites#bootstrapping-the-client) using the injected script. Open the `/assets/custom.js` file and add the following code at the top of the file, replacing `<LAUNCHDARKLY_CLIENT_ID>` with your LaunchDarkly project's client ID.

```javascript
const client = LDClient.initialize(
  "<LAUNCHDARKLY_CLIENT_ID>",
  {
    key: "anonymous",
  },
  {
    bootstrap: window.ldFlags,
  }
);
```

Since the flag values are automatically synced between LaunchDarkly and the KV store, every time this page is served, it will automatically have the current flag state values injected before the page is sent to the end user. This has the effect of eliminating any flash of content that can be caused even by a very brief rendering delay between when the UI is initially displayed and flag values are received by the LaunchDarkly client that are used to update the user interface.

Once you've bootstrapped the client, you can use the LaunchDarkly SDK client as you normally would. For example, the following code when added to `/assets/custom.js` initializes the SDK, gets the value of a boolean flag named `show-about-us` and then calls the `showAboutUs()` method to either hide or display the "About Us" section of the home page.

```javascript
const client = LDClient.initialize("61409b046ca8d52601d179ef", {
  key: "anonymous",
});
client.on("ready", async function () {
  const showAboutUs = await client.variation("show-about-us", false);
  displayAboutUs(showAboutUs);
});

client.on("change", async function () {
  const showAboutUs = await client.variation("show-about-us", false);
  displayAboutUs(showAboutUs);
});
```

Since the value of `show-about-us` is bootstrapped in the client, there is no latency when getting the initial flag value and displaying the content to the user. In addition, because the code watches for the `change` event, any changes to the flag once the page is loaded will be reflected immediately.

## Modifying content at the edge

Cloudflare Workers can be used to modify the content being served to an end user at the CDN level, before they ever receive it in their browser client. This can be useful to do things like [A/B testing](https://developers.cloudflare.com/workers/examples/ab-testing), [fetching a HTML or API response](https://developers.cloudflare.com/workers/examples/fetch-html), targeting content based upon user data or personalizing content for an authenticated user. 

The below example shows how to use the value of a string flag to replace the header text on a page. This type of solution could be modified for use in A/B testing, for slowly rolling out content changes or for personalizing content.

The first thing you need to do is create a string flag in LaunchDarkly as shown in the example below that has two variations containing the header text. You do not need to enable it for client-side SDKs since you'll be calling it within a Worker function.

![Create a string flag in LaunchDarkly](CF-SDK-create-string-flag.png)

Within the Worker, you can reuse the LaunchDarkly client you created in the prior example. However, since you'll be getting flag values in multiple places, it can be easier to create a single function to handle asynchronously retrieving the flag values. The following function allows you to pass in a user `key` and a `user` obect and returns the value of the flag for that user. If no user is passed, it defaults to `anonymous` as a user key. This function can be placed anywhere within the `/workers-site/index.js` Worker file that is outside an existing code block. For instance, just prior to the `addEventListener` function block.

```javascript
async function getFlagValue(key, user) {
  let flagValue;
  if (!user) {
    user = {
      key: "anonymous",
    };
  }
  flagValue = await ldClient.variation(key, user, false);
  return flagValue;
}
```

Next create an element handler to modify the DOM element that you would like to populate with the returned value of the string flag in LaunchDarkly. In the below case, the element handler will be called for every `<h1>` element within the page HTML. You'll need to use the same instance of `HTMLRewriter` as from the above example as trying to create more than one instance of `HTMLRewriter` in a single Worker will cause errors. Place the following line immediately after the `rewriter.on()` call from the prior example.

```javascript
rewriter.on("h1", new H1ElementHandler());
```

Lastly, the element handler gets the value of the flag named `header-text` and replaces the text within the `<h1>` tag with the result. Place this code following the existing `FlagsStateInjector` block from the prior example.

```javascript
class H1ElementHandler {
  async element(element) {
    // replace the header text with the value of a string flag
    const headerText = await getFlagValue("header-text");
    element.setInnerContent(headerText);
  }
}
```

The result is shown in the below video clip. Note that because the change happens within the Worker, the page will need to be refreshed to reflect any flag change after the user has already received the page source.

<video controls width="100%">
    <source src="header-values.mp4"
            type="video/mp4">
    Sorry, your browser doesn't support embedded videos.
</video>

## Modifying the Response Headers for a Response

Modifying the response headers for a response can be a powerful tool. It can be used to change existing headers for testing purposes, to add custom headers that your code can respond to or even redirect a user to a different page. In this example, you'll use a JSON flag value in LaunchDarkly to create an object containing the custom headers you want added to the response using a Cloudflare Worker.

The first thing you need to do is create a JSON flag in LaunchDarkly. JSON flags can contain any valid, arbitrary JSON data. In the example shown below, the flag contains an array of request header names and values. In one case, a `x-launchdarkly-hello` header is set, while in the other it is not.

![Creating a JSON flag in LaunchDarkly](CF-SDK-json-flag.png)

Next, you'll need to get the flag value. Since the result of getting the flag is an array of objects, the code below loops through each item in the array and sets a header for the response for each item found in the array. This code should go within the `handleEvent` function prior to the `return` within the `try` block.

```javascript
// allow headers to be altered
const response = new Response(page.body, page);

const customHeader = await getFlagValue("custom-response-headers");
customHeader.headers.forEach((header) => {
  response.headers.set(header.name, header.value);
});
```

As shown highlighted in the image below, when the flag is turned on, the user will receive a `x-launchdarkly-hello` response header with the value of "Hello from LaunchDarkly".

![Viewing custom response headers](CF-SDK-custom-headers.png)

## Conclusion

Cloudflare Workers offer an great way to deploy serverless code "to the edge", meaning they are deployed to a CDN and served to your end users from the CDN servers closest to their location. This makes them incredibly fast and a great way to perform all sorts of logic and processing on the user's request and response as it is in flight. If you're wondering what else you can do with Cloudflare Workers, check out there [list of examples](https://developers.cloudflare.com/workers/examples), including things like sending a [conditional response](https://developers.cloudflare.com/workers/examples/conditional-response) or [rewriting links](https://developers.cloudflare.com/workers/examples/rewrite-links).

Combining Workers with LaunchDarkly feature management is a powerful combination, offering you the ability to bootstrap client-side flags with zero latency or allowing you to control how your code runs at the edge in Cloudflare simply by flipping a flag.

