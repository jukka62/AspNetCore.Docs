---
title: ASP.NET Core Blazor state management
author: guardrex
description: Learn how to persist state in Blazor server-side apps.
monikerRange: '>= aspnetcore-3.0'
ms.author: riande
ms.custom: mvc
ms.date: 07/17/2019
uid: blazor/state-management
---
# ASP.NET Core Blazor state management

By [Steve Sanderson](https://github.com/SteveSandersonMS)

Blazor server-side is a stateful app framework. Most of the time, the app maintains an ongoing connection to the server, and the user's state is held in the server's memory in what's known as a *circuit*. 

Examples of state held for a user's circuit include:

* The rendered UI&mdash;the hierarchy of component instances and their most recent render output.
* The values of any fields and properties in component instances.
* Data held in [dependency injection (DI)](xref:fundamentals/dependency-injection) service instances that are scoped to the circuit.

> [!NOTE]
> This article addresses state persistence in Blazor server-side apps. Blazor client-side apps can take advantage of [client-side state persistence in the browser](#client-side-in-the-browser) but require custom solutions or 3rd party packages beyond the scope of this article.

## Blazor circuits

Occasionally, users may experience a temporary network connection loss, after which Blazor attempts to reconnect users to their original circuits so they can continue to use the app.

However, reconnecting users to their original circuit in the server's memory isn't always possible:

* The server can't retain disconnected circuits forever. The server must release disconnected circuits after a timeout or when the server is under memory pressure. The timeout and retention limits are configurable.
* In multiserver, load-balanced deployment environments, any server processing requests may no longer be available at any given time. Servers may fail or be torn down when no longer required to handle the overall volume of requests.
* The user might close and reopen their browser or reload the page, which removes any state held in the browser's memory, such as values set through JavaScript interop calls.

When a user can't be reconnected to their original circuit, the user receives a new circuit with an empty state. This is equivalent to closing and re-opening a desktop app.

## Preserve state across circuits

In some scenarios, preserving state across circuits is desirable. If a user is adding items to a shopping cart, an app can retain the shopping cart's contents if the web server becomes unavailable and the user's browser is forced to start a new circuit with a new web server. In general, maintaining state across circuits applies to scenarios where users are actively creating data, not simply reading data that already exists.

To preserve state beyond a single circuit, *don't merely store the data in the server's memory*. The app must persist the data to some other storage location. State persistence isn't automatic&mdash;you must take steps when developing the app to implement stateful data persistence.

Not all state in a circuit requires persistence. Data persistence is typically only required for high-value state that users have expended effort to create. For example, persist state for the contents of a complex multistep webform or for a commercially important component of an app, such as a shopping cart that represents potential revenue. It's usually *not* necessary to preserve easily-recreated state, such as the username entered into a sign-in dialog that hasn't been submitted.

> [!IMPORTANT]
> An app can only persist *app state*. UIs can't be persisted, such as component instances and their render trees. Components and render trees aren't generally serializable. To persist something similar to UI state, such as the expanded nodes of a tree view in the UI, the app must have custom code to model the behavior as serializable app state.

## Where to persist state

Three common locations exist for persisting state in a Blazor server-side app. Each approach is best suited to different scenarios and has different caveats.

### Server-side in a database

For permanent data persistence or for any data that must span multiple users or devices, an independent server-side database is almost certainly the best choice. Options include:

* Relational SQL database
* Key-value store
* Blob store
* Table store

After data is saved in the database, a new circuit can be started by a user at any time. The user's data is retained and available in any new circuit.

For more information on Azure data storage options, see the [Azure Storage Documentation](/azure/storage/) and [Azure Databases](https://azure.microsoft.com/product-categories/databases/).

### URL

For transient data representing navigation state, model the data as a part of the URL. Examples of state modeled in the URL include:

* The ID of a viewed entity.
* The current page number in a paged grid.

The contents of the browser's address bar are retained:

* If the user manually reloads the page.
* If the web server becomes unavailable&mdash;the user is forced to reload the page in order to connect to a different server.

For information on defining URL patterns with the `@page` directive, see <xref:blazor/routing>.

### Client-side in the browser

For transient data that the user is actively creating, a common backing store is the browser's `localStorage` and `sessionStorage` collections. Client-side storage scenarios have an advantage over server-side storage in that the app isn't required to manage or clear the stored state if the circuit is abandoned.

> [!NOTE]
> "Client-side" in this section refers to client-side scenarios in the browser, not the [Blazor client-side hosting model](xref:blazor/hosting-models#client-side). `localStorage` and `sessionStorage` can be used in Blazor client-side apps but only by writing custom code or using a 3rd party package. The scenarios explained in this article apply to the [Blazor server-side hosting model](xref:blazor/hosting-models#server-side).

`localStorage` and `sessionStorage` differ as follows:

* `localStorage` is scoped to the user's browser. If the user reloads the page or closes and re-opens the browser, the state persists. If the user opens multiple browser tabs, the state is shared across the tabs. Data persists in `localStorage` until explicitly cleared.
* `sessionStorage` is scoped to the user's browser tab. If the user reloads the tab, the state persists. If the user closes the tab or the browser, the state is lost. If the user opens multiple browser tabs, each tab has its own independent version of the data.

Generally, using `sessionStorage` is safer. `sessionStorage` avoids the risk with `localStorage` that a user opens multiple tabs and encounters bugs or confusing behavior when a tab overwrites the state of other tabs. However, `localStorage` is the better choice if the app must persist state across closing and re-opening the browser.

Caveats for using browser storage:

* Similar to use of a server-side database, loading and saving are asynchronous.
* Unlike a server-side database, storage isn't available during prerendering because the requested page doesn't exist in the browser during the prerendering stage.
* Storage of a few kilobytes of data is reasonable to persist for Blazor server-side apps. Beyond a few kilobytes, you must consider the performance implications because the data is loaded and saved across the network.
* Users may view or tamper with the data. Risks of tampering can be mitigated for Blazor server-side apps using [Data Protection](xref:security/data-protection/introduction).

## Third-party browser storage solutions

Third-party NuGet packages provide APIs for working with `localStorage` and `sessionStorage` in Blazor apps.

It's worth considering choosing a package that transparently uses ASP.NET Core's [Data Protection](xref:security/data-protection/introduction) scenarios to encrypt the stored data and reduce the potential risk of tampering with stored data. If JSON-serialized data is stored in plaintext, users can not only see that data (for example, using browser developer tools) but also modify the stored data arbitrarily. Securing data isn't always a problem because the data might be trivial in nature (for example, setting the color of a UI element background). Allowing inspection or tampering of sensitive data can be a problem.

## Protected Browser Storage experimental package

> [!WARNING]
> `Microsoft.AspNetCore.ProtectedBrowserStorage` is an unsupported experimental package unsuitable for production use at this time.

An example of a NuGet package that provides [Data Protection](xref:security/data-protection/introduction) for `localStorage` and `sessionStorage` is [Microsoft.AspNetCore.ProtectedBrowserStorage](https://www.nuget.org/packages/Microsoft.AspNetCore.ProtectedBrowserStorage). Currently, this is an unsupported experimental package unsuitable for production use.

### Installation

To install the `Microsoft.AspNetCore.ProtectedBrowserStorage` package:

1. In the Blazor server-side app project, add a package reference to [Microsoft.AspNetCore.ProtectedBrowserStorage](https://www.nuget.org/packages/Microsoft.AspNetCore.ProtectedBrowserStorage).
1. In the top-level HTML (for example, in the *Pages/_Host.cshtml* file in the default project template), add the following `<script>` tag:

   ```html
   <script src="_content/Microsoft.AspNetCore.ProtectedBrowserStorage/protectedBrowserStorage.js"></script>
   ```

1. In the `Startup.ConfigureServices` method, call `AddProtectedBrowserStorage` to add `localStorage` and `sessionStorage` services to the service collection:

   ```csharp
   services.AddProtectedBrowserStorage();
   ```

### Save and load data within a component

In any component that requires loading or saving data to browser storage, use `@inject` to inject an instance of either `ProtectedLocalStorage` or `ProtectedSessionStorage`, depending on which backing store you wish to use. In the following example, `sessionStorage` is used:

```cshtml
@using Microsoft.AspNetCore.ProtectedBrowserStorage
@inject ProtectedSessionStorage ProtectedSessionStore
```

The `@using` statement can be placed into an *_Imports.razor* file instead, which makes the namespace available to larger segments of the app or the whole app.

Whenever the user performs an action that requires data storage, the service is used to store the data in `sessionStorage`.

To persist the `currentCount` value in the `Counter` component of the project template, modify the `IncrementCount` method to use `ProtectedSessionStore.SetAsync`:

```csharp
private async Task IncrementCount()
{
    currentCount++;
    await ProtectedSessionStore.SetAsync("count", currentCount);
}
```

In larger, more realistic apps, storage of individual fields is an unlikely scenario. Apps are more likely to store entire model objects that include complex state. `ProtectedSessionStore` automatically JSON serializes and deserializes any data provided.

In the preceding code example, the `currentCount` data is stored as `sessionStorage['count']` in the user's browser. If `sessionStorage['count']` is evaluated in the browser's developer console, the data isn't stored in plaintext but rather is protected using ASP.NET Core's [Data Protection](xref:security/data-protection/introduction).

To recover the `currentCount` data if the user returns to the `Counter` component later (including if they're on an entirely new circuit), use `ProtectedSessionStore.GetAsync`:

```csharp
protected override async Task OnInitializedAsync()
{
    currentCount = await ProtectedSessionStore.GetAsync<int>("count");
}
```

If the component's parameters include navigation state, call `ProtectedSessionStore.GetAsync` and assign the result in `OnParametersSetAsync`, not `OnInitializedAsync`. `OnInitializedAsync` is only called once when the component is first instantiated and isn't called again later if the user navigates to a different URL while remaining on the same page.

> [!WARNING]
> The examples in this section only work if the server doesn't have prerendering enabled. With prerendering enabled, an error is generated similar to:
>
> > JavaScript interop calls cannot be issued at this time. This is because the component is being prerendered.
>
> Either disable prerendering or add additional code to work with prerendering, as described in the [Handle prerendering](#handle-prerendering) section.

### Handle the loading state

Since browser storage is asynchronous (accessed over a network connection), there's always a period of time before the data is loaded and available for use by a component. For the best results, render a loading-state message while loading is in progress instead of displaying blank or default data.

One approach is to track whether the data is `null` (still loading) or not. In the default `Counter` component, the count is held in an `int`. Make `currentCount` nullable by adding a question mark (`?`) to the type (`int`):

```csharp
private int? currentCount;
```

Instead of displaying the count and **Increment** button unconditionally, choose to display these elements only if the data is loaded:

```cshtml
@if (currentCount.HasValue)
{
    <p>Current count: <strong>@currentCount</strong></p>

    <button @onclick="IncrementCount">Increment</button>
}
else
{
    <p>Loading...</p>
}
```

### Handle prerendering

During prerendering, an interactive connection to the user's browser doesn't exist, and the browser doesn't yet have any page in which it can run JavaScript. It's not possible to interact with `localStorage` or `sessionStorage` at that time. If the page attempts to interact with storage, an error is generated similar to:

> JavaScript interop calls cannot be issued at this time. This is because the component is being prerendered*.

One way to resolve the error is to disable prerendering. That's often the best choice if the app makes heavy use of browser-based storage. Prerendering adds complexity and wouldn't benefit the app because the app can't prerender any useful content until `localStorage` or `sessionStorage` are available.

To disable prerendering, open the *Pages/_Host.cshtml* file and remove the call to `Html.RenderComponentAsync`.
Open the `Startup.cs` file, and replace the call to `endpoints.MapBlazorHub()` with `endpoints.MapBlazorHub<App>("app")`. `App` is the type of the root component. `"app"` is a CSS selector specifying the location for the root component.

To keep prerendering enabled, perhaps because it's useful on other pages that don't use `localStorage` or `sessionStorage`, defer the loading operation until the browser is connected to the circuit. The following is an example for storing a counter value:

```cshtml
@inject ProtectedLocalStorage ProtectedLocalStore
@inject IComponentContext ComponentContext

... rendering code goes here ...

@code {
    private int? currentCount;
    private bool isWaitingForConnection;

    protected override async Task OnInitAsync()
    {
        if (ComponentContext.IsConnected)
        {
            // Looks like the app isn't prerendering, so the data can be immediately
            // loaded from browser storage.
            await LoadStateAsync();
        }
        else
        {
            // Prerendering is in progress, so the app defers the load operation
            // until later.
            isWaitingForConnection = true;
        }
    }

    protected override async Task OnAfterRenderAsync()
    {
        // By this stage, the client has connected back to the server, and
        // browser services are available. If the app didn't load the data earlier,
        // the app should do so now and then trigger a new render.
        if (isWaitingForConnection)
        {
            isWaitingForConnection = false;
            await LoadStateAsync();
            StateHasChanged();
        }
    }

    private async Task LoadStateAsync()
    {
        currentCount = await ProtectedLocalStore.GetAsync<int>("prerenderedCount");
    }

    private async Task IncrementCount()
    {
        currentCount++;
        await ProtectedSessionStore.SetAsync("count", currentCount);
    }
}
```

### Factor out the state preservation to a common location

If many components rely on browser-based storage, re-implementing the preceding pattern many times creates code duplication, especially if the app deals with the complexity of working with prerendering. One option for avoiding code duplication is to create a *state provider parent component* that encapsulates the required logic. Child components can work with persisted data without regard to the state persistence mechanism.

In the following example of a `CounterStateProvider` component, counter data is persisted:

```cshtml
@using Microsoft.AspNetCore.ProtectedBrowserStorage
@inject ProtectedSessionStorage ProtectedSessionStore

@if (hasLoaded)
{
    <CascadingValue Value="@this">
        @ChildContent
    </CascadingValue>
}
else
{
    <p>Loading...</p>
}

@code {
    [Parameter]
    public RenderFragment ChildContent { get; set; }

    public int CurrentCount { get; set; }
    private bool hasLoaded;

    protected override async Task OnInitAsync()
    {
        CurrentCount = await ProtectedSessionStore.GetAsync<int>("count");
        hasLoaded = true;
    }

    public async Task SaveChangesAsync()
    {
        await ProtectedSessionStore.SetAsync("count", CurrentCount);
    }
}
```

The `CounterStateProvider` component handles the loading phase by not rendering its child content until loading is complete.

To use the `CounterStateProvider` component, wrap an instance of the component around any other component that requires access to the counter state. To make the state accessible to all components in an app, wrap the `CounterStateProvider` component around the `Router` in the `App` component (*App.razor*):

```cshtml
<CounterStateProvider>
    <Router AppAssembly="typeof(Startup).Assembly">
        ...
    </Router>
</CounterStateProvider>
```

Wrapped components receive and can modify the persisted counter state. The following `Counter` component implements the pattern:

```cshtml
@page "/counter"

<p>Current count: <strong>@CounterStateProvider.CurrentCount</strong></p>

<button @onclick="IncrementCount">Increment</button>

@code {
    [CascadingParameter]
    private CounterStateProvider CounterStateProvider { get; set; }

    private async Task IncrementCount()
    {
        CounterStateProvider.CurrentCount++;
        await CounterStateProvider.SaveChangesAsync();
    }
}
```

This preceding component isn't required to interact with `ProtectedBrowserStorage`, nor does it deal with any "loading" phase.

To deal with prerendering as described earlier, `CounterStateProvider` can be amended so that all of the components that consume the counter data automatically work with prerendering with no further code changes. See the [Handle prerendering](#handle-prerendering) section for details.

In general, it's a good idea to follow this *state provider parent component* pattern:

* To consume state in many other components.
* If there's just one top-level state object to persist.

To persist many different state objects and consume different subsets of them in different places, it's better to avoid handling the loading and saving of state globally, which avoids loading or saving irrelevant data.
