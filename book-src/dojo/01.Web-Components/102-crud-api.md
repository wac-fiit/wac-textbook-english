# API for record modification

---

>info:>
Template for pre-created container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-101`

---

In this section, we will continue with the definition of the API specification and modification of the component for record editing, allowing our users to work with data in the application. We will still use only experimental simulation of the WEB API.

1. Open the file `${WAC_ROOT}/ambulance-ufe/api/ambulance-wl.openapi.yaml` and add additional operations to the `paths` section:

```yaml
...
paths:
  "/waiting-list/{ambulanceId}/entries":
  ...
  "/waiting-list/{ambulanceId}/entries/{entryId}":       @_add_@
    get:        @_add_@
      tags:       @_add_@
        - ambulanceWaitingList       @_add_@
      summary: Provides details about waiting list entry       @_add_@
      operationId: getWaitingListEntry       @_add_@
      description: >-       @_add_@
        By using ambulanceId and entryId you can details of particular entry       @_add_@
        item ambulance.       @_add_@
      parameters:       @_add_@
        - in: path       @_add_@
          name: ambulanceId       @_add_@
          description: pass the id of the particular ambulance       @_add_@
          required: true       @_add_@
          schema:       @_add_@
            type: string       @_add_@
        - in: path       @_add_@
          name: entryId       @_add_@
          description: pass the id of the particular entry in the waiting list       @_add_@
          required: true       @_add_@
          schema:       @_add_@
            type: string       @_add_@
      responses:       @_add_@
        "200":       @_add_@
          description: value of the waiting list entries       @_add_@
          content:       @_add_@
            application/json:       @_add_@
              schema:       @_add_@
                $ref: "#/components/schemas/WaitingListEntry"       @_add_@
              examples:       @_add_@
                response:       @_add_@
                  $ref: "#/components/examples/WaitingListEntryExample"       @_add_@
        "404":       @_add_@
          description: Ambulance or Entry with such ID does not exists       @_add_@
    put:       @_add_@
      tags:       @_add_@
        - ambulanceWaitingList       @_add_@
      summary: Updates specific entry       @_add_@
      operationId: updateWaitingListEntry       @_add_@
      description: Use this method to update content of the waiting list entry.       @_add_@
      parameters:       @_add_@
        - in: path       @_add_@
          name: ambulanceId       @_add_@
          description: pass the id of the particular ambulance       @_add_@
          required: true       @_add_@
          schema:       @_add_@
            type: string       @_add_@
        - in: path       @_add_@
          name: entryId       @_add_@
          description: pass the id of the particular entry in the waiting list       @_add_@
          required: true       @_add_@
          schema:       @_add_@
            type: string       @_add_@
      requestBody:       @_add_@
        content:       @_add_@
          application/json:       @_add_@
            schema:       @_add_@
              $ref: "#/components/schemas/WaitingListEntry"       @_add_@
            examples:       @_add_@
              request:       @_add_@
                $ref: "#/components/examples/WaitingListEntryExample"       @_add_@
        description: Waiting list entry to update       @_add_@
        required: true       @_add_@
      responses:       @_add_@
        "200":       @_add_@
          description: >-       @_add_@
            value of the waiting list entry with re-computed estimated time of       @_add_@
            ambulance entry       @_add_@
          content:       @_add_@
            application/json:       @_add_@
              schema:       @_add_@
                $ref: "#/components/schemas/WaitingListEntry"       @_add_@
              examples:       @_add_@
                response:       @_add_@
                  $ref: "#/components/examples/WaitingListEntryExample"       @_add_@
        "403":       @_add_@
          description: >-       @_add_@
            Value of the entryID and the data id is mismatching. Details are       @_add_@
            provided in the response body.       @_add_@
        "404":       @_add_@
          description: Ambulance or Entry with such ID does not exists       @_add_@
    delete:       @_add_@
      tags:       @_add_@
        - ambulanceWaitingList       @_add_@
      summary: Deletes specific entry       @_add_@
      operationId: deleteWaitingListEntry       @_add_@
      description: Use this method to delete the specific entry from the waiting list.       @_add_@
      parameters:       @_add_@
        - in: path       @_add_@
          name: ambulanceId       @_add_@
          description: pass the id of the particular ambulance       @_add_@
          required: true       @_add_@
          schema:       @_add_@
            type: string       @_add_@
        - in: path       @_add_@
          name: entryId       @_add_@
          description: pass the id of the particular entry in the waiting list       @_add_@
          required: true       @_add_@
          schema:       @_add_@
            type: string       @_add_@
      responses:       @_add_@
        "204":       @_add_@
          description: Item deleted       @_add_@
        "404":       @_add_@
          description: Ambulance or Entry with such ID does not exists       @_add_@
```

We have added a new endpoint `/waiting-list/{ambulanceId}/entries/{entryId}` for retrieving the record details, updating, and deleting it. Correspondingly, we have named the operations and adapted the responses. In the case of the `PUT` method, we have also added a `requestBody` that will contain the updated version of the record we want to update.

Now, in the directory `${WAC_ROOT}/ambulance-ufe`, run the command to generate the API access code:

```ps
  npm run openapi
```

2. Open the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.tsx` and add the code that will fetch the record from the API server and handle the data correctness:

```tsx
...
import { AmbulanceWaitingListApiFactory, WaitingListEntry } from '../../api/ambulance-wl'; @_add_@
...
export class <Pfx>AmbulanceWlEditor {

  @Prop() entryId: string;
  @Prop() ambulanceId: string; @_add_@
  @Prop() apiBase: string;  @_add_@

  @Event({eventName: "editor-closed"}) editorClosed: EventEmitter<string>;
  @State() private duration = 15;

  @State() entry: WaitingListEntry;  @_add_@
  @State() errorMessage:string;  @_add_@
  @State() isValid: boolean;  @_add_@

  private formElement: HTMLFormElement;  @_add_@

  private async getWaitingEntryAsync(): Promise<WaitingListEntry> {   @_add_@
      if ( !this.entryId ) {   @_add_@
        this.isValid = false;   @_add_@
        return undefined   @_add_@
      }   @_add_@
      try {   @_add_@
        const response  @_add_@
            = await AmbulanceWaitingListApiFactory(undefined, this.apiBase)  @_add_@
              .getWaitingListEntry(this.ambulanceId, this.entryId)   @_add_@
          @_add_@
        if (response.status < 299) {   @_add_@
            this.entry = response.data;   @_add_@
            this.isValid = true;   @_add_@
        } else {   @_add_@
            this.errorMessage = `Cannot retrieve list of waiting patients: ${response.statusText}`   @_add_@
        }   @_add_@
      } catch (err: any) {   @_add_@
        this.errorMessage = `Cannot retrieve list of waiting patients: ${err.message || "unknown"}`   @_add_@
      }   @_add_@
      return undefined;   @_add_@
  }   @_add_@

  async componentWillLoad() {  @_add_@
      this.getWaitingEntryAsync();  @_add_@
  }  @_add_@

  ...
```

The functionality is similar to the `getWaitingListAsync` method in the `<pfx>-ambulance-wl-list` component. In this case, however, we load only one record, the one we want to edit. Note that the `componentWillLoad()` method does not use _async/await_. As a result, the component first renders - empty - and the fields are populated only after obtaining data from the API. That's why we have also marked the `entry` field with the `@State()` decorator. The behavior during the loading of the list and the editor will be slightly different. In the first case, the UI is blocked until the data is retrieved, and in the second case, the UI is rendered without data, and the data from the API is added later. Neither solution is ideal; our code should be supplemented with the `dataLoading` state, and during this state, we should display a corresponding indicator, such as the text "Loading data...". We leave this change for your independent work.

Next, in the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.tsx`, modify the `render()` function:

```tsx
...
render() {
  if(this.errorMessage) { @_add_@
      return ( @_add_@
      <Host> @_add_@
        <div class="error">{this.errorMessage}</div> @_add_@
      </Host> @_add_@
      ) @_add_@
  } @_add_@
  return (
      <Host>
        <form ref={el => this.formElement = el}> @_add_@
          <md-filled-text-field label="Meno a Priezvisko" 
            required value={this.entry?.name} @_add_@
            oninput={ (ev: InputEvent) => {  @_add_@
              if(this.entry) {this.entry.name = this.handleInputEvent(ev)}  @_add_@
            } }>  @_add_@
            <md-icon slot="leading-icon">person</md-icon>
          </md-filled-text-field>

          <md-filled-text-field label="Registračné číslo pacienta" 
            required value={this.entry?.patientId} @_add_@
            oninput={ (ev: InputEvent) => { @_add_@
              if(this.entry) {this.entry.patientId = this.handleInputEvent(ev)}  @_add_@
            } }>  @_add_@
            <md-icon slot="leading-icon">fingerprint</md-icon>
          </md-filled-text-field>

          <md-filled-text-field label="Čakáte od" disabled 
            value={this.entry?.waitingSince}> @_add_@
            <md-icon slot="leading-icon">watch_later</md-icon>
          </md-filled-text-field>

          <md-filled-select label="Dôvod návštevy" 
            value={this.entry?.condition?.code} @_add_@
            oninput = { (ev: InputEvent) => { @_add_@
              if(this.entry) {this.entry.condition.code = this.handleInputEvent(ev)} @_add_@
            } }>  @_add_@
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
        </form> @_add_@

        <div class="duration-slider">
          <span class="label">Predpokladaná doba trvania:&nbsp; </span>
          <span class="label">{this.duration}</span>
          <span class="label">&nbsp;minút</span>
          <md-slider
            min="2" max="45" value={this.entry?.estimatedDurationMinutes || 15} ticks labeled @_add_@
            oninput={ (ev:InputEvent) => { @_add_@
              if(this.entry) { @_add_@
              this.entry.estimatedDurationMinutes  @_add_@
                  = Number.parseInt(this.handleInputEvent(ev))}; @_add_@
              this.handleSliderInput(ev) @_add_@
            } }></md-slider> @_add_@
        </div>

        <md-divider inset></md-divider>

        <div class="actions">
          <md-filled-tonal-button id="delete" disabled={ !this.entry }  @_important_@
            onClick={() => this.deleteEntry()} >  @_important_@
            <md-icon slot="icon">delete</md-icon>
            Zmazať
          </md-filled-tonal-button>
          <span class="stretch-fill"></span>
          <md-outlined-button id="cancel"
            onClick={() => this.editorClosed.emit("cancel")}>
            Zrušiť
          </md-outlined-button>
          <md-filled-button id="confirm" disabled={ !this.isValid }  @_important_@
            onClick={() => this.updateEntry() }  @_important_@
            >
            <md-icon slot="icon">save</md-icon>
            Uložiť
          </md-filled-button>
        </div>
      </Host>
    );
}  
```

With this change, we set the values of individual input fields and modified the reactions to value changes. The `Delete` and `Save` buttons are active only when a record is loaded and valid. Notice how we obtained a reference to the `form` element using the `ref` attribute. Now, let's add methods to handle the events we used in the `render()` method:

```tsx
...
render() {
  ...
}

private handleInputEvent( ev: InputEvent): string {  @_add_@
  const target = ev.target as HTMLInputElement;  @_add_@
  // check validity of elements  @_add_@
  this.isValid = true;  @_add_@
  for (let i = 0; i < this.formElement.children.length; i++) {  @_add_@
      const element = this.formElement.children[i]  @_add_@
      if ("reportValidity" in element) {  @_add_@
      const valid = (element as HTMLInputElement).reportValidity();  @_add_@
      this.isValid &&= valid;  @_add_@
      }  @_add_@
  }  @_add_@
  return target.value  @_add_@
}  @_add_@

private async updateEntry() {      @_add_@
  try {  @_add_@
      const response = await AmbulanceWaitingListApiFactory(undefined, this.apiBase) @_add_@
        .updateWaitingListEntry(this.ambulanceId, this.entryId, this.entry)  @_add_@
      if (response.status < 299) {  @_add_@
        this.editorClosed.emit("store")  @_add_@
      } else {  @_add_@
        this.errorMessage = `Cannot store entry: ${response.statusText}`  @_add_@
      }  @_add_@
    } catch (err: any) {  @_add_@
      this.errorMessage = `Cannot store entry: ${err.message || "unknown"}`  @_add_@
    }        @_add_@
}  @_add_@

private async deleteEntry() { @_add_@
  try { @_add_@
      const response = await AmbulanceWaitingListApiFactory(undefined, this.apiBase) @_add_@
        .deleteWaitingListEntry(this.ambulanceId, this.entryId) @_add_@
      if (response.status < 299) { @_add_@
      this.editorClosed.emit("delete") @_add_@
      } else { @_add_@
      this.errorMessage = `Cannot delete entry: ${response.statusText}` @_add_@
      } @_add_@
  } catch (err: any) { @_add_@
      this.errorMessage = `Cannot delete entry: ${err.message || "unknown"}` @_add_@
  } @_add_@
} @_add_@

```

In the `handleInputEvent()` method, we added validation for individual input fields. We used the [`reportValidity()`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/reportValidity) method, which checks the constraints set on our element and, in case of an invalid value, displays an error message near the element. This method also updates the current value in the `entry` object.

The `updateEntry()` and `deleteEntry()` methods call the corresponding methods from the API client.

3. Modify the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.css` and adjust the component styles to:


```css
:host {
  --_wl-editor_gap: var(--wl-gap, 0.5rem);
  width: 100%;  @_add_@
  height: 100%;  @_add_@

  display: flex;  @_remove_@
  flex-direction: column;  @_remove_@
  gap: var(--_wl-editor_gap);  @_remove_@
  padding: var(--_wl_editor_gap);  @_remove_@
}

form{  @_add_@
  display: flex;  @_add_@
  flex-direction: column;    @_add_@
  gap: var(--_wl-editor_gap);  @_add_@
  padding: var(--_wl-editor_gap);  @_add_@
  }  @_add_@

.error {  @_add_@
  margin: auto;  @_add_@
  color: red;  @_add_@
  font-size: 2rem;  @_add_@
  font-weight: 900;  @_add_@
  text-align: center;  @_add_@
  }  @_add_@

.duration-slider {
  ...
```

4. Modify the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-app/<pfx>-ambulance-wl-app.tsx`, in the `render()` function, add attributes for the `<pfx>--ambulance-wl-editor` element:

```tsx
...
render() {
  ...
  return (
      { element === "editor" 
      ? <<pfx>-ambulance-wl-editor entry-id={entryId}
        ambulance-id={this.ambulanceId} api-base={this.apiBase} @_add_@
        oneditor-closed={ () => navigate("./list")}
      ></<pfx>-ambulance-wl-editor>
      : <<pfx>-ambulance-wl-list  ambulance-id={this.ambulanceId} api-base={this.apiBase}
        onentry-clicked={ (ev: CustomEvent<string>)=> navigate("./entry/" + ev.detail) } >
        </<pfx>-ambulance-wl-list>
      }
      </Host>
      );
...
}
```

5. The last modification will be made in the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-list/<pfx>-ambulance-wl-list.tsx`. Set the correct id when selecting an item from the list:

```tsx
...
render() {
  ...
  {this.waitingPatients.map((patient) => @_important_@
    <md-list-item onClick={ () => this.entryClicked.emit(patient.id)}> @_important_@
      <div slot="headline">{patient.name}</div>
      <div slot="supporting-text">{"Predpokladaný vstup: " + this.isoDateToLocale(patient.estimatedStart)}</div>
        <md-icon slot="start">person</md-icon>
    </md-list-item>
  ...
```

6. In the directory `${WAC_ROOT}/ambulance-ufe`, start the development server with the command:

```ps
npm run start
```

and in the browser, go to the page [http://localhost:3333](http://localhost:3333) to verify the functionality. We still use only simulated data, so our data remains unchanged after refreshing, and when selecting an item, we see only a sample record. In the browser, open _Developer tools_ (_F12_) and go to the _Network_ tab to monitor communication with the API server.

7. We still need to handle loading the list of possible issues - `conditions` for the dropdown list and creating a new record. Again, start by modifying the API specification in the file `${WAC_ROOT}/ambulance-ufe/api/ambulance-wl.openapi.yaml`. Add the following entry to the main `tags` section:


```yaml
...
tags:
  - name: ambulanceWaitingList
    description: Ambulance Waiting List API
  - name: ambulanceConditions @_add_@
    description: Patient conditions and symptoms handled in the ambulance @_add_@
...
```

and in the `paths` section, add the following operations:

```yaml
...
paths:
  "/waiting-list/{ambulanceId}/entries":
    get:
      ...
    post:  @_add_@
      tags: @_add_@
        - ambulanceWaitingList  @_add_@
      summary: Saves new entry into waiting list @_add_@
      operationId: createWaitingListEntry @_add_@
      description: Use this method to store new entry into the waiting list. @_add_@
      parameters: @_add_@
        - in: path @_add_@
          name: ambulanceId @_add_@
          description: pass the id of the particular ambulance @_add_@
          required: true @_add_@
          schema: @_add_@
            type: string @_add_@
      requestBody: @_add_@
        content: @_add_@
          application/json: @_add_@
            schema: @_add_@
              $ref: "#/components/schemas/WaitingListEntry" @_add_@
            examples: @_add_@
              request-sample:  @_add_@
                $ref: "#/components/examples/WaitingListEntryExample" @_add_@
        description: Waiting list entry to store @_add_@
        required: true @_add_@
      responses: @_add_@
        "200": @_add_@
          description: >- @_add_@
            Value of the waiting list entry with re-computed estimated time of @_add_@
            ambulance entry @_add_@
          content: @_add_@
            application/json: @_add_@
              schema: @_add_@
                $ref: "#/components/schemas/WaitingListEntry" @_add_@
              examples: @_add_@
                updated-response:  @_add_@
                  $ref: "#/components/examples/WaitingListEntryExample" @_add_@
        "400": @_add_@
          description: Missing mandatory properties of input object. @_add_@
        "404": @_add_@
          description: Ambulance with such ID does not exists @_add_@
        "409": @_add_@
          description: Entry with the specified id already exists @_add_@
  "/waiting-list/{ambulanceId}/entries/{entryId}":
    ...
  "/waiting-list/{ambulanceId}/condition": @_add_@
    get: @_add_@
      tags: @_add_@
        - ambulanceConditions @_add_@
      summary: Provides the list of conditions associated with ambulance @_add_@
      operationId: getConditions @_add_@
      description: By using ambulanceId you get list of predefined conditions @_add_@
      parameters: @_add_@
        - in: path @_add_@
          name: ambulanceId @_add_@
          description: pass the id of the particular ambulance @_add_@
          required: true @_add_@
          schema: @_add_@
            type: string @_add_@
      responses: @_add_@
        "200": @_add_@
          description: value of the predefined conditions @_add_@
          content: @_add_@
            application/json: @_add_@
              schema: @_add_@
                type: array @_add_@
                items: @_add_@
                  $ref: "#/components/schemas/Condition" @_add_@
              examples: @_add_@
                response: @_add_@
                  $ref: "#/components/examples/ConditionsListExample" @_add_@
        "404": @_add_@
          description: Ambulance with such ID does not exists   @_add_@
...
```

For the `GET` operation for the new endpoint `/waiting-list/{ambulanceId}/condition`, which returns a list of possible issues, we used the new tag `ambulanceConditions`. This will result in generating a new class and [factory functions](https://en.wikipedia.org/wiki/Factory_method_pattern). Finally, add a new response example `ConditionsListExample`:

```yaml
...
components:
  schemas:
  ...
  examples:
    ...
    ConditionsListExample:  @_add_@
      summary: Sample of GP ambulance conditions  @_add_@
      description: |  @_add_@
        Example list of possible conditions, symptoms, and visit reasons  @_add_@
      value:  @_add_@
        - value: Teploty  @_add_@
          code: subfebrilia  @_add_@
          reference: "https://zdravoteka.sk/priznaky/zvysena-telesna-teplota/"  @_add_@
          typicalDurationMinutes: 20  @_add_@
        - value: Nevoľnosť  @_add_@
          code: nausea  @_add_@
          reference: "https://zdravoteka.sk/priznaky/nevolnost/"  @_add_@
          typicalDurationMinutes: 45  @_add_@
        - value: Kontrola  @_add_@
          code: followup  @_add_@
          typicalDurationMinutes: 15  @_add_@
        - value: Administratívny úkon  @_add_@
          code: administration  @_add_@
          typicalDurationMinutes: 10  @_add_@
        - value: Odber krvi  @_add_@
          code: blood-test  @_add_@
          typicalDurationMinutes: 10  @_add_@
```

8. Save the changes and in the directory `${WAC_ROOT}/ambulance-ufe`, run the command:

```ps
npm run openapi
```

9. To distinguish creating a new record from editing an existing one in the editor, we will use a special reserved id `@new`. Modify the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.tsx`:

```tsx
...
export class <Pfx>AmbulanceWlEditor {
  ...
  private async getWaitingEntryAsync(): Promise<WaitingListEntry> {
      if(this.entryId === "@new") {  @_add_@
        this.isValid = false;  @_add_@
        this.entry = {  @_add_@
          id: "@new",  @_add_@
          patientId: "",  @_add_@
          waitingSince: "",  @_add_@
          estimatedDurationMinutes: 15  @_add_@
        };  @_add_@
        return this.entry;  @_add_@
      }  @_add_@
      ...
  }

  render() {
      ...
      <md-filled-tonal-button id="delete" disabled={!this.entry || this.entry?.id === "@new" } @_important_@
      ...
  }

  private async updateEntry() {
      try {
        const response = await AmbulanceWaitingListApiFactory(undefined, this.apiBase) @_remove_@
            .updateWaitingListEntry(this.ambulanceId, this.entryId, this.entry)  @_remove_@
        // store or update
        const api = AmbulanceWaitingListApiFactory(undefined, this.apiBase);  @_add_@
        const response   @_add_@
            = this.entryId === "@new"   @_add_@
            ? await api.createWaitingListEntry(this.ambulanceId, this.entry)  @_add_@
            : await api.updateWaitingListEntry(this.ambulanceId, this.entryId, this.entry);  @_add_@
        ...
  }
  ...
```

In this version, if we find that the `entryId` parameter is set to the value `@new`, instead of loading the `entry` from the API, we will create a new record. When saving the record, we will use the corresponding method of our API client. The `delete` button will be disabled for the scenario of creating a new record.

Now, modify the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-list/<pfx>-ambulance-wl-list.tsx` and add a control for creating a new record:


```tsx
...
render() {
return (
  <Host>
    {this.errorMessage
      ...
    }
    <md-filled-icon-button class="add-button"  @_add_@
      onclick={() => this.entryClicked.emit("@new")}>  @_add_@
      <md-icon>add</md-icon>  @_add_@
    </md-filled-icon-button>  @_add_@
  </Host>
  );
}
```

We used a new element from the `@material/web` library, so add the following to the file `${WAC_ROOT}/ambulance-ufe/src/global/app.ts`:

```tsx
import '@material/web/iconbutton/filled-icon-button' @_add_@
...
```

We will also modify the style for the new element in the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-list/<pfx>-ambulance-wl-list.css`:

```css
:host {
  display: block;
  width: 100%;
  height: 100%;  
  position: relative; @_add_@
  padding-bottom: 2rem; @_add_@
}

.error {
  ...
}

.add-button {  @_add_@
  position: absolute;  @_add_@
  right: 1rem;  @_add_@
  bottom: 0;  @_add_@
  --md-filled-icon-button-container-size: 4rem;  @_add_@
  --md-filled-icon-button-icon-size: 3rem;  @_add_@
}  @_add_@
```

The _+_ button is positioned at the bottom right of our element. Since we are using `position: absolute`, we need to set `position: relative` for the `<pfx>-ambulance-wl-list` element; otherwise, the button's position would be adjusted relative to the page's position and size or relative to the position of the nearest ancestor whose position is relative.

You can test the functionality; you should see the list with the _+_ button, and clicking the button should display the editor for creating a new record.

![List with the Add Entry button](./img/102-01-AddEntry.png)

10. The list of visit reasons is currently static, and we want to achieve having a specific list for each ambulance. Open the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.tsx` and modify it:


```tsx
import { AmbulanceConditionsApiFactory, AmbulanceWaitingListApiFactory, Condition, WaitingListEntry } from '../../api/ambulance-wl'; @_important_@
...
export class <Pfx>AmbulanceWlEditor {  
  ... 
  @State() entry: WaitingListEntry;
  @State() conditions: Condition[];  @_add_@
  ...

  private async getWaitingEntryAsync(): Promise<WaitingListEntry> {
      ...
  }

  private async getConditions(): Promise<Condition[]> {  @_add_@
      try {  @_add_@
        const response = await AmbulanceConditionsApiFactory(undefined, this.apiBase).getConditions(this.ambulanceId);  @_add_@
        if (response.status < 299) {  @_add_@
        this.conditions = response.data;  @_add_@
        }  @_add_@
      } catch (err: any) {  @_add_@
        // no strong dependency on conditions  @_add_@
      }  @_add_@
      // always have some fallback condition  @_add_@
      return this.conditions || [{  @_add_@
        code: "fallback",  @_add_@
        value: "Neurčený dôvod návštevy",  @_add_@
        typicalDurationMinutes: 15,  @_add_@
      }];  @_add_@
  }  @_add_@

  componentWillLoad() {
      this.getWaitingEntryAsync();
      this.getConditions(); @_add_@
  }  
```

We added loading the list of visit reasons for the ambulance. In case the loading fails, we use a fallback value that is sufficient for the normal functioning of the ambulance. Now, let's modify the way we display the list of visit reasons in the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.tsx`:

```tsx
...
render() {
  ....
  return (
      ....
      <md-filled-select label="Dôvod návštevy"  @_remove_@
            ... @_remove_@
      </md-filled-select> @_remove_@
      <!-- pre prehľadnosť použijeme pomocnú metódu -->
      {this.renderConditions()} @_add_@
    </form>
    ...
  );
}

private renderConditions() {   @_add_@
  let conditions = this.conditions || [];   @_add_@
  // we want to have this.entry`s condition in the selection list   @_add_@
  if (this.entry?.condition) {   @_add_@
    const index = conditions.findIndex(condition => condition.code === this.entry.condition.code)   @_add_@
    if (index < 0) {   @_add_@
    conditions = [this.entry.condition, ...conditions]   @_add_@
    }   @_add_@
  }   @_add_@
  return (   @_add_@
    <md-filled-select label="Dôvod návštevy"   @_add_@
      display-text={this.entry?.condition?.value}   @_add_@
      oninput={(ev: InputEvent) => this.handleCondition(ev)} >   @_add_@
    <md-icon slot="leading-icon">sick</md-icon>   @_add_@
    {this.entry?.condition?.reference ? @_add_@
      <md-icon slot="trailing-icon" class="link"   @_add_@
        onclick={()=> window.open(this.entry.condition.reference, "_blank")}>   @_add_@
          open_in_new   @_add_@
      </md-icon>   @_add_@
    : undefined   @_add_@
    }   @_add_@
    {conditions.map(condition => {   @_add_@
        return (   @_add_@
          <md-select-option   @_add_@
          value={condition.code} @_add_@
          selected={condition.code === this.entry?.condition?.code}> @_add_@
              <div slot="headline">{condition.value}</div> @_add_@
          </md-select-option> @_add_@
        )   @_add_@
    })}   @_add_@
    </md-filled-select>   @_add_@
  );   @_add_@
} @_add_@


private handleCondition(ev: InputEvent) {  @_add_@
  if(this.entry) {  @_add_@
    const code = this.handleInputEvent(ev)   @_add_@
    const condition = this.conditions.find(condition => condition.code === code);   @_add_@
    this.entry.condition = Object.assign({}, condition);   @_add_@
    this.entry.estimatedDurationMinutes = condition.typicalDurationMinutes;   @_add_@
    this.duration = condition.typicalDurationMinutes;   @_add_@
  }  @_add_@
}  @_add_@
```

We moved the rendering and control of the visit reasons list to a separate method. In the `renderConditions()` method, we first check if we have the visit reason of the current record in the list of visit reasons. If not, we add it to the beginning of the list. Then, we render the list of visit reasons and set the current record's value as selected. In the `handleCondition()` method, we ensure that when selecting a specific visit reason, the expected duration of the visit is appropriately adjusted. The user can further modify this time. Additionally, we allowed the user to familiarize themselves with the description of the visit reason before the actual visit. For this, we added the `open_in_new` icon, which opens a new window with the description of the visit reason. The effect of this element could be, for example, that the patient is already informed about what to expect upon entering, and the entire examination can proceed more efficiently.

11. Verify the functionality. If your development server is not running, run the command in the directory `${WAC_ROOT}/ambulance-ufe`:

```ps
npm run start
```

and open the page in the browser [http://localhost:3333](http://localhost:3333). Keep in mind that we are still using only the simulated WEB API.

12. We will expand the editor tests with a simple test for the correctness of the _Name and Surname_ field. Open the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/test/<pfx>-ambulance-wl-editor.spec.tsx` and modify it:


```tsx
import { newSpecPage } from '@stencil/core/testing';
import { <Pfx>AmbulanceWlEditor } from '../<pfx>-ambulance-wl-editor'; 
import axios from "axios";  @_add_@
import MockAdapter from "axios-mock-adapter";  @_add_@
import { Condition, WaitingListEntry } from '../../../api/ambulance-wl';  @_add_@

describe('<pfx>-ambulance-wl-editor', () => { 
  const sampleEntry: WaitingListEntry = {  @_add_@
      id: "entry-1",  @_add_@
      patientId: "p-1",  @_add_@
      name: "Juraj Prvý",  @_add_@
      waitingSince: "20240203T12:00",  @_add_@
      estimatedDurationMinutes: 20,  @_add_@
      condition: {  @_add_@
        "value": "Nevoľnosť",  @_add_@
        "code": "nausea",  @_add_@
        "reference": "https://zdravoteka.sk/priznaky/nevolnost/"  @_add_@
      }  @_add_@
  };  @_add_@
  @_add_@
  const sampleConditions: Condition[] = [  @_add_@
      {  @_add_@
        "value": "Teploty",  @_add_@
        "code": "subfebrilia",  @_add_@
        "reference": "https://zdravoteka.sk/priznaky/zvysena-telesna-teplota/",  @_add_@
        "typicalDurationMinutes": 20  @_add_@
      },  @_add_@
      {  @_add_@
        "value": "Nevoľnosť",  @_add_@
        "code": "nausea",  @_add_@
        "reference": "https://zdravoteka.sk/priznaky/nevolnost/",  @_add_@
        "typicalDurationMinutes": 45  @_add_@
      },  @_add_@
  ];  @_add_@
  @_add_@
  let delay = async (miliseconds: number) => await new Promise<void>(resolve => {  @_add_@
        setTimeout(() => resolve(), miliseconds);  @_add_@
  })  @_add_@
    @_add_@
  let mock: MockAdapter;  @_add_@
  @_add_@
  beforeAll(() => { mock = new MockAdapter(axios); });  @_add_@
  afterEach(() => { mock.reset(); });  @_add_@

  it('buttons shall be of different type', async () => {
      mock.onGet(/^.*\/entries\/.+/).reply(200, sampleEntry);  @_add_@
      mock.onGet(/^.*\/condition$/).reply(200, sampleConditions);  @_add_@

      const page = await newSpecPage({
        components: [<Pfx>AmbulanceWlEditor], 
        html: `<<pfx>-ambulance-wl-editor entry-id="test-entry"   @_important_@
            ambulance-id="test-ambulance" api-base="http://sample.test/api">  @_important_@
        </<pfx>-ambulance-wl-editor>`,  @_important_@
      });
      await delay(300); @_add_@
      await page.waitForChanges(); @_add_@
      let items: any = await page.root.shadowRoot.querySelectorAll("md-filled-button");
      expect(items.length).toEqual(1);
      ...
  });

  it('first text field is patient name', async () => {           @_add_@
      mock.onGet(/^.*\/entries\/.+/).reply(200, sampleEntry);  @_add_@
      mock.onGet(/^.*\/condition$/).reply(200, sampleConditions);  @_add_@
      @_add_@
      const page = await newSpecPage({  @_add_@
        components: [PfxAmbulanceWlEditor],  @_add_@
        html: `<<pfx>-ambulance-wl-editor entry-id="test-entry" ambulance-id="test-ambulance" api-base="http://sample.test/api"></<pfx>-ambulance-wl-editor>`,  @_add_@
      });  @_add_@
      let items: any = await page.root.shadowRoot.querySelectorAll("md-filled-text-field");  @_add_@
      @_add_@
      await delay(300);  @_add_@
      await page.waitForChanges();  @_add_@
        @_add_@
      expect(items.length).toBeGreaterThanOrEqual(1);  @_add_@
      expect(items[0].getAttribute("value")).toEqual(sampleEntry.name);  @_add_@
  });  @_add_@

});
```

In this case, we simulate responses from two different endpoints, so when calling the `onGet()` method, we use a regular expression to distinguish individual requests. Also, in this case, we must assume that when the `newSpecPage` function is called, the initial rendering has not yet completed the asynchronous data loading. By calling `await delay(300)`, we allow the asynchronous call to be realized and processed. Subsequently, by calling `await page.waitForChanges()`, we ensure that all necessary updates of elements are executed, and then we verify the resulting state. As in previous cases, we provide only an example test, and the development of further tests is left to your independent work.

Verify the functionality of the tests by running the command:

```ps
npm run test
```

13. Archive the code changes:

```ps
git add .
git commit -m "Ambulance waiting list CRUD operations"
git push
```
