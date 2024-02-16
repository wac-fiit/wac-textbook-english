# Editor of Items and Navigation

---

>info:>
Template for a pre-created container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-090`

---

After the initial steps that will help us automate our work during further development, we will now proceed with implementing functionality for our micro-application. In this section, we will address adding new items and navigation between them.

## Component for Editing Records

1. Create a new component for our editor. Navigate to the `${WAC_ROOT}/ambulance-ufe` directory and execute the following command:

```ps
npm run generate
```

Choose `<pfx>-ambulance-wl-editor` as the item name, and for the subsequent questions, select the default options. In the directory `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor`, we now have a pre-prepared template for our new component.

2. In the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.tsx`, modify the `render` method as follows:

```tsx

render() {
  return (
    <Host>
      <md-filled-text-field label="Meno a Priezvisko" >
        <md-icon slot="leading-icon">person</md-icon>
      </md-filled-text-field>

      <md-filled-text-field label="Registračné číslo pacienta" >
        <md-icon slot="leading-icon">fingerprint</md-icon>
      </md-filled-text-field>

      <md-filled-text-field label="Čakáte od" disabled>
        <md-icon slot="leading-icon">watch_later</md-icon>
      </md-filled-text-field>

      <md-filled-select label="Dôvod návštevy">
        <md-icon slot="leading-icon">sick</md-icon>
        <md-select-option value="folowup">
          <div slot="headline">Kontrola</div>
        </md-select-option>
        <md-select-option value="nausea">
          <div slot="headline">Nevoľnosť</div>
        </md-select-option>
        <md-select-option value="fever">
          <div slot="headline">Horúčka</div>
        </md-select-option>
        <md-select-option value="ache-in-throat">
          <div slot="headline">Bolesti hrdla</div>
        </md-select-option>
      </md-filled-select>

      <div class="duration-slider">
        <span class="label">Predpokladaná doba trvania:&nbsp; </span>
        <span class="label">{this.duration}</span>
        <span class="label">&nbsp;minút</span>
        <md-slider
          min="2" max="45" value={this.duration} ticks labeled
          oninput={this.handleSliderInput.bind(this)}></md-slider>
      </div>

      <md-divider></md-divider>
      <div class="actions">
        <md-filled-tonal-button id="delete"
          onClick={() => this.editorClosed.emit("delete")}>
          <md-icon slot="icon">delete</md-icon>
          Zmazať
        </md-filled-tonal-button>
        <span class="stretch-fill"></span>
        <md-outlined-button id="cancel"
          onClick={() => this.editorClosed.emit("cancel")}>
          Zrušiť
        </md-outlined-button>
        <md-filled-button id="confirm"
          onClick={() => this.editorClosed.emit("store")}>
          <md-icon slot="icon">save</md-icon>
          Uložiť
        </md-filled-button>
      </div>
    </Host>
  );
}
```

and add the following code to the `<Pfx>AmbulanceWlEditor` class:

```tsx
import { Component, Host, Prop, State, h, EventEmitter, Event } from '@stencil/core'; @_add_@
...
export class <Pfx>AmbulanceWlEditor {

  @Prop() entryId: string; @_add_@
  @_add_@
  @Event({eventName: "editor-closed"}) editorClosed: EventEmitter<string>; @_add_@
  @_add_@
  @State() private duration = 15 @_add_@
  @_add_@
  private handleSliderInput(event: Event) {  @_add_@
    this.duration = +(event.target as HTMLInputElement).value;  @_add_@
  } @_add_@
...
```

In the code, notice the variable `duration` and how, when the value of the `md-slider` element changes, we set this property to the current value. Using the `@Prop` or `@State` decorator ensures that our element will be redrawn with the current values when their values change. `@Prop() entryId` declares that our element will contain an attribute named `entry-id`, which will determine the identifier of the record we want to edit. Property names are automatically transformed into HTML attributes in `lower-case` format (details can be found [here](https://stenciljs.com/docs/properties#attribute-name-attribute)). Furthermore, we declare that our element will emit events of type `EventEmitter<string>` named `editor-closed`.

3. In the code, we used new components from the `@material/web` library. Open the file `${WAC_ROOT}/ambulance-ufe/src/global/app.ts` and add the import of the relevant components:

```ts
import '@material/web/list/list'  
import '@material/web/list/list-item'   
import '@material/web/icon/icon'
import '@material/web/textfield/filled-text-field'  @_add_@
import '@material/web/select/filled-select'  @_add_@
import '@material/web/select/select-option'  @_add_@
import '@material/web/slider/slider'  @_add_@
import '@material/web/button/filled-button'  @_add_@
import '@material/web/button/filled-tonal-button'  @_add_@
import '@material/web/button/outlined-button'  @_add_@
import '@material/web/divider/divider'  @_add_@
...
```

4. In the file `${WAC_ROOT}/ambulance-ufe/src/index.html`, modify the HTML body to the following form:

```html
...
<body>
      <<pfx>-ambulance-wl-list></<pfx>-ambulance-wl-list>
      <<pfx>-ambulance-wl-editor></<pfx>-ambulance-wl-editor> @_add_@
</body>
...
```

Execute the command in the directory `${WAC_ROOT}/ambulance-ufe`:

```ps
npm run start
```

Open the page [http://localhost:3333/](http://localhost:3333/) in your browser. You should see both of our components, with the `<pfx>-ambulance-wl-editor` component appearing unordered for now.

5. Modify the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.css`:


```css
:host {
  --_wl-editor_gap: var(--wl-gap, 0.5rem);

  display: flex;
  flex-direction: column;  
  gap: var(--_wl-editor_gap);
  padding: var(--_wl-editor_gap);
}

.duration-slider {
  display: flex;
  flex-direction: row;
  align-content: space-around;
  align-items: center;
}

.actions {
  display: flex;
  flex-direction: row;
  justify-content: flex-end;
  gap: 1rem;

}

.stretch-fill {
  flex: 10 0 0;
}

md-divider {
  margin-bottom: var(--_wl-editor_gap);
} 
```

Using CSS Flexbox, we arranged individual elements in our component and allowed setting the space between them using a custom CSS variable `--wl-gap`. After saving and compiling, you should see the following result:

![Editor Display](./img/090-01-Editor.png)

>info:> To achieve a responsive design for your page, it is recommended to leverage [CSS Flexbox](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_flexible_box_layout/Basic_concepts_of_flexbox) and [CSS Grid](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_grid_layout/Basic_concepts_of_grid_layout) to the maximum extent, along with using relative units such as `rem`, `em`, `vw`, and `vh`.

Check the functionality of individual elements. Currently, both components are designed, and now we need to implement the application logic. Firstly, we want each of these components to be displayed on separate pages (in the context of a Single Page Application - meaning without loading a page from the web server). For this, we'll need a new component that, depending on the current page address, will display the corresponding component.

## Navigation between Components

Currently, we display both the list of waiting patients and the editor for individual records on one page. Our goal is to have the list displayed at [http://localhost:3333/list](http://localhost:3333/list) and the editor at [http://localhost:3333/entry/<id-item>](http://localhost:3333/ambulance-wl/entry/0). This is to enable navigation between these elements for the user and, at the same time, preserve the functionality of navigation buttons. Additionally, we want our application to function even when located at a different server root address, such as `http://wac-hospital.wac/<pfx>-ambulance/`. For this, we'll need a new component whose task is to display one of our components depending on the current page address and respond to changes in navigation.

For navigation purposes, we'll use the [Navigation API], which aims to allow single-page applications to gain control over requests to change the page and, if necessary, prevent or allow loading a new page. However, as of now _(August 2023)_, this API is not available for all browsers. We'll assist by creating a simplified [polyfill](https://developer.mozilla.org/en-US/docs/Glossary/Polyfill) implementation. While this implementation is not fully-featured, it is sufficient for our needs.

>info:> Ensuring the connection of the page address - URL of individual items - with the current record is an important concept in the development of Single Page Web Applications. It allows users to create bookmarks to which they can return later, or share a URL pointing to a specific record.

1. Create a new file `${WAC_ROOT}/ambulance-ufe/src/global/navigation.ts` and insert the following code into it:

```ts
class PolyNavigationDestination {
    constructor(url: string) {
        this.url = url;
    }
    url: string;
}

class PolyNavigateEvent extends Event {
    constructor(destination: string | URL, info?: any) {
        super('navigate', { bubbles: true, cancelable: true });
        
        let rebased  = new URL(destination,  document.baseURI)
        this.canIntercept = location.protocol === rebased.protocol
        && location.host === rebased.host && location.port === rebased.port;
        this.destination = new PolyNavigationDestination(rebased.href);
        this.info = info;
    }

    destination: PolyNavigationDestination;
    canIntercept: boolean = true;
    info: any
    isIntercepted = false;

    intercept(_options?: any ) {
        this.isIntercepted = true;
        // options are ignored in this implementation, e.g. no handler or scroll
    }

    scroll(_options?: any ) {
        // not implemented 
    }
}
```

This code defines two new classes that we will use to represent [navigation events](https://developer.mozilla.org/en-US/docs/Web/API/NavigateEvent). These classes will be used in case the browser does not support the [Navigation API].

Next, add the following code to the file:

```ts
...
declare global {
  interface Window { navigation: any; }
}

export function registerNavigationApi() {
  if (!window.navigation) { // provide pollyfill only if not present @_important_@
      // simplified version of navigation api
      window.navigation = new EventTarget();
      const oldPushState = window.history.pushState.bind(window.history);

      window.history.pushState = (f => function pushState() {
          var ret = f.apply(this, arguments);
          let url = arguments[2];
          window.navigation.dispatchEvent(new PolyNavigateEvent(url));
          return ret;
      })(window.history.pushState);

      window.addEventListener("popstate", () => {
          window.navigation.dispatchEvent(new PolyNavigateEvent(document.location.href));
      });

      let previousUrl = '';
      const observer = new MutationObserver(function () {
          if (location.href !== previousUrl) {
              previousUrl = location.href;
              window.navigation.dispatchEvent(new PolyNavigateEvent(location.href));
          }
      });

      const config = { subtree: true, childList: true };
      observer.observe(document, config);
      window.onunload = () => {
          observer.disconnect();
      }

      window.navigation.navigate = (
          url: string, 
          options: {state?: any; info?: any; history?: "auto" | "replace" | "push";}
      ) => {
        const ev = new PolyNavigateEvent(url, options?.info);
        window.navigation.dispatchEvent(ev);
        if (ev.isIntercepted) {
            oldPushState(options?.state || {}, '', url);
        } else {
            window.open(url, "_self");
        }
      }

      window.navigation.back = (
        _options?: {info?: any;}
      ) => {
          window.history.back();
          return {
              commited: Promise.resolve(),
              finished: new Promise<void>( resolve => setTimeout( () => resolve()  , 0))
          }
      }
  }
}
```

This function first checks if the browser already supports the [Navigation API]. If not, it overloads the [`window.history.pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState) method. This method is commonly used in SPA applications to achieve a similar purpose as the Navigation API. Next, we register for the [`popstate`] event, which is triggered when the "Back" navigation button is pressed in the browser. Finally, we register a [`MutationObserver`](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) to observe changes in the `href` value of the [Location](https://developer.mozilla.org/en-US/docs/Web/API/Location). This covers all the ways that may indicate a change in the address during the activity of our application in the browser. In all cases, we generate a `PolyNavigateEvent` event, simulating the functionality of the Navigation API.

Additionally, we have created a [`navigate`](https://developer.mozilla.org/en-US/docs/Web/API/Navigation/navigate) function that allows triggering navigation in our application. However, this function is merely a wrapper for calling `window.history.pushState` and generating a `PolyNavigateEvent` event. We have not implemented other methods and events of the Navigation API because we do not need them in our application. Similarly, the [`back`](https://developer.mozilla.org/en-US/docs/Web/API/Navigation/back) function is implemented in a similar way.

Finally, modify the file `${WAC_ROOT}/ambulance-ufe/src/global/app.ts` and add the appropriate code loading.

```ts
import { registerNavigationApi } from './navigation.js' @_add_@

export default function() { 
    registerNavigationApi() @_add_@
}
```

2. Generate a new component `<pfx>-ambulance-wl-app` by executing the following command in the directory `${WAC_ROOT}/ambulance-ufe`:

```ps
npm run generate
```

In the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-app/<pfx>-ambulance-wl-app.tsx`, modify the `render` method to the following format:

```tsx
render() {
  let element = "list"
  let entryId = "@new"

  if ( this.relativePath.startsWith("entry/"))
  {
    element = "editor";
    entryId = this.relativePath.split("/")[1]
  }

  const navigate = (path:string) => {
    const absolute = new URL(path, new URL(this.basePath, document.baseURI)).pathname;
    window.navigation.navigate(absolute)
  }

  return (
    <Host>
      { element === "editor" 
      ? <<pfx>-ambulance-wl-editor entry-id={entryId}
          oneditor-closed={ () => navigate("./list")} >
        </<pfx>-ambulance-wl-editor>
      : <<pfx>-ambulance-wl-list></<pfx>-ambulance-wl-list>
      }
      
    </Host>
  );
}
```

This method displays one of our two components based on the value of the `relativePath` class property. If the page address is in the form of `editor/<id>`, it will display the `<pfx>-ambulance-wl-editor` component and set its `entry-id` attribute to the value `<id>`. This attribute will serve to identify the record we want to edit. If the page address is in the form of `list` or any other form, it will display the `<pfx>-ambulance-wl-list` component. In the case of the editor, we register an event handler for the `onEditor-closed` event, which is triggered when the "Cancel" or "Save" button is pressed in the `<pfx>-ambulance-wl-editor` component. In this case, we call the `window.navigation.navigate()` function, which triggers navigation to the waiting list.

> Note: In principle, when closing the editor, we could also call the `window.navigation.back()` method, which would allow us to return to the previous page. However, this could lead to inconsistent behavior in the application, as we might end up returning to a page that is not the waiting list. Therefore, it is better to use the `window.navigation.navigate()` method, which explicitly allows navigation to the waiting list.

Furthermore, in the same file, modify the `<Pfx>AmbulanceWlApp` class to include the following code:

```tsx
import { Component, Host, Prop, State, h } from '@stencil/core'; @_important_@
@_add_@
declare global {@_add_@
  interface Window { navigation: any; } @_add_@
}
...
export class <Pfx>AmbulanceWlApp { 

  @State() private relativePath = ""; @_add_@
  @_add_@
  @Prop() basePath: string=""; @_add_@
  @_add_@
  componentWillLoad() { @_add_@
    const baseUri = new URL(this.basePath, document.baseURI || "/").pathname; @_add_@
    @_add_@
    const toRelative = (path: string) => { @_add_@
      if (path.startsWith( baseUri)) { @_add_@
        this.relativePath = path.slice(baseUri.length) @_add_@
      } else { @_add_@
        this.relativePath = "" @_add_@
      } @_add_@
    } @_add_@
    @_add_@
    window.navigation?.addEventListener("navigate", (ev: Event) => { @_add_@
      if ((ev as any).canIntercept) { (ev as any).intercept(); }  @_add_@
      let path = new URL((ev as any).destination.url).pathname; @_add_@
      toRelative(path);   @_add_@
    }); @_add_@
    @_add_@
    toRelative(location.pathname) @_add_@
  } @_add_@

  render() {
    ...
```

In the `componentWillLoad()` method, we registered an event handler for the `navigate` event, which is triggered during navigation in our application. By calling the `intercept()` method, we informed the browser that we handle URL changes from our code. In this handler, we then obtained the path to the current page and stored it in the `relativePath` property. This property will be used to decide which of our components to display. The `base-path` element attribute also allows us to deploy our application on a server with a different root address, or more nested than the `baseUri` of the document.

> Warning: Not all applications will want to call `intercept()` for any URL. This is specific to our exercise. If we did not call `intercept()`, the browser would attempt to load the page from the web server.

An important aspect is working with `document.baseURI`. This property determines the base URL of our document (i.e., the page) against which the relative path is then determined. Under normal circumstances, this address is the same as the address from which we loaded our page, i.e., the address initially entered into the browser. Because in the case of single-page applications, this address may differ from the actual application address. For example, if we entered the address [http://localhost:3333/editor/id](http://localhost:3333/editor/id) in the browser, our code would not work correctly, or it would have to implicitly assume on which path this application is actually deployed. For proper functionality, it is therefore necessary to set the `baseUri` of the document in the case of single-page applications and possibly allow the configuration of this value through environment variables if we plan to deliver our components as a standalone HTTP server.

To make the necessary change, modify the file `${WAC_ROOT}/ambulance-ufe/src/index.html` by adding the `<base>` element to the `<head>` section:

```html
<!DOCTYPE html>
<html dir="ltr" lang="en">
  <head>
    <base href="/"/> @_add_@
....
```

Then, in the same file, use our new component `<pfx>-ambulance-wl-app`:

```html
<body>
  <<pfx>-ambulance-wl-list></<pfx>-ambulance-wl-list> @_remove_@
  <<pfx>-ambulance-wl-editor></<pfx>-ambulance-wl-editor> @_remove_@
  <!-- web application root component -->
  <<pfx>-ambulance-wl-app base-path="/ambulance-wl/"></<pfx>-ambulance-wl-app> @_add_@
  
</body>
```

4. If it is not active, start the development web server (`npm run start`) and navigate to the page [http://localhost:3333/ambulance-wl](http://localhost:3333/ambulance-wl) in your browser, where you will see the component with the waiting list. Then, go to the page [http://localhost:3333/ambulance-wl/entry/0](http://localhost:3333/ambulance-wl/entry/0), where you will see the component for editing individual records. After pressing any of the buttons, you will return to the list of waiting patients.

5. To complete our interaction, modify the file `${WAC_ROOT}/ambulance-ufe/src/components/pfx-ambulance-wl-list/pfx-ambulance-wl-list.tsx` and add the following code sections:

```tsx
import { Component, Event, EventEmitter,  Host, h } from '@stencil/core'; @_important_@
...
export class PfxAmbulanceWlList { 

@Event({ eventName: "entry-clicked"}) entryClicked: EventEmitter<string>; @_add_@
...
render() {
return (
  <Host>
    <md-list>   
      {this.waitingPatients.map((patient, index) =>   @_important_@
        <md-list-item onClick={ () => this.entryClicked.emit(index.toString())}> @_add_@
            <div slot="headline">{patient.name}</div>
            <div slot="supporting-text">{"Predpokladaný vstup: " + this.isoDateToLocale(patient.estimatedStart)}</div>
            <md-icon slot="start">person</md-icon>
        </md-list-item>
    ...
```

This code ensures that when clicking on a record in the list of waiting patients, the `entry-clicked` event will be triggered, containing the index of the record we want to edit. We will handle this event in the `<pfx>-ambulance-wl-app` component and trigger navigation to the editor page. Modify the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-app/<pfx>-ambulance-wl-app.tsx` and add the following code sections:


```tsx
...
render() {
  ...

  return (
    ...
      : <<pfx>-ambulance-wl-list 
          onentry-clicked={ (ev: CustomEvent<string>)=> navigate("./entry/" + ev.detail) } > @_add_@
        </<pfx>-ambulance-wl-list>
    ...
```

In the application, you can now click on an item in the list, triggering navigation to the editor page.

> Note: We intentionally handle navigation only in the `<pfx>-ambulance-wl-app.tsx` element. The waiting list element can be used in different contexts, and clicking on an item may trigger an alternative response without the need to further modify the waiting list element itself. This achieves reusability of the `<pfx>-ambulance-wl-list` element in other contexts. Similarly, the editor element can be reused in other contexts.

6. Modify the test files so that we can submit our changes to the project. Open the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-app/test/<pfx>-ambulance-wl-app.spec.tsx` and modify it as follows:


```ts
import { newSpecPage } from '@stencil/core/testing';
import { <Pfx>AmbulanceWlApp } from '../<pfx>-ambulance-wl-app';

describe('<pfx>-ambulance-wl-app', () => {

  it('renders editor', async () => {
    const page = await newSpecPage({
      url: `http://localhost/entry/@new`,
      components: [<Pfx>AmbulanceWlApp],
      html: `<<pfx>-ambulance-wl-app base-path="/"></<pfx>-ambulance-wl-app>`,
    });
    page.win.navigation = new EventTarget()
    const child = await page.root.shadowRoot.firstElementChild;
    expect(child.tagName.toLocaleLowerCase()).toEqual ("<pfx>-ambulance-wl-editor");
    
  });

  it('renders list', async () => {
    const page = await newSpecPage({
      url: `http://localhost/ambulance-wl/`,
      components: [<Pfx>AmbulanceWlApp],
      html: `<<pfx>-ambulance-wl-app base-path="/ambulance-wl/"></<pfx>-ambulance-wl-app>`,
    });
    page.win.navigation = new EventTarget()
    const child = await page.root.shadowRoot.firstElementChild;
    expect(child.tagName.toLocaleLowerCase()).toEqual("<pfx>-ambulance-wl-list");
  });
});
```

Next, modify the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/test/<pfx>-ambulance-wl-editor.spec.tsx`.

```ts
import { newSpecPage } from '@stencil/core/testing';
import { <Pfx>AmbulanceWlEditor } from '../<pfx>-ambulance-wl-editor';

describe('<pfx>-ambulance-wl-editor', () => {
  it('buttons shall be of different type', async () => {
    const page = await newSpecPage({
      components: [<Pfx>AmbulanceWlEditor],
      html: `<<pfx>-ambulance-wl-editor entry-id="@new"></<pfx>-ambulance-wl-editor>`,
    });
    let items: any = await page.root.shadowRoot.querySelectorAll("md-filled-button");
    expect(items.length).toEqual(1);
    items = await page.root.shadowRoot.querySelectorAll("md-outlined-button");
    expect(items.length).toEqual(1);

    items = await page.root.shadowRoot.querySelectorAll("md-filled-tonal-button");
    expect(items.length).toEqual(1);
  });
});
```

7. Archive the changes and synchronize with the remote repository:

```ps
git add .
git commit -m "entry editor and navigation"
git push
```

8. The last change is to modify the web component that should be displayed in the micro Front-End application. For this, we just need to adjust the configuration. Open the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/webcomponent.yaml` and in the `navigation` section, enter the new name of the element:

```yaml
navigation:
- element: <pfx>-ambulance-wl-list @_remove_@

# aplication context element
- element: <pfx>-ambulance-wl-app    @_add_@
```

Archive the changes with the commands in the directory `${WAC_ROOT}/ambulance-gitops`:

```ps
git add .
git commit -m "entry editor and navigation"
git push
```

If your [Docker Desktop] cluster is not active, start it. After a while, when the [Flux CD] operator applies all changes - updating the image version, deploying configuration changes, go to [http://localhost:30331](http://localhost:30331), and then select the application _Waiting List ..._. You should see the page with the `<pfx>-ambulance-wl-list` component. Clicking on a record should display the record editor. After pressing the "Cancel" button, you should return to the waiting list. Although we did not explicitly set the `base-path` attribute in the file `${WAC_ROOT}/ambulance-gitops/apps/<pfx>-ambulance-ufe/webcomponent.yaml`, this attribute is automatically added to the element by the `ufe-controller` service based on the value of the `path` attribute.

> Build Circle: In case of issues with applying configuration changes, you can check the status of individual objects in the cluster with the following commands:
>
> ```ps
> kubectl -n wac-hospital get gitrepository 
> kubectl -n wac-hospital get kustomization 
> kubectl -n wac-hospital get imagerepository
> kubectl -n wac-hospital get imagepolicy
> kubectl -n wac-hospital get imageupdateautomation
> ```
>
> Alternatively, use similar commands, but replace `get` with `describe` for a more detailed output. We recommend having the [OpenLens] tool installed and using it for more efficient analysis of issues in the cluster.
