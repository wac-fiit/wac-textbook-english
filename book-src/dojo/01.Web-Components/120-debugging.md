# Debugging the Application

---

>info:>
Template for a pre-built container ([Details here](../99.Problems-Resolutions/01.development-containers.md)):
`registry-1.docker.io/milung/wac-ufe-120`

---

In this step, we will demonstrate how to debug the application using the _Developer Tools_.

1. We will test debugging by adding a new patient. In the file `${WAC_ROOT}/ambulance-ufe/src/components/ambulance-wl-editor/ambulance-wl-editor.tsx`, modify the getWaitingEntryAsync` function as follows:

```tsx
private async getWaitingEntryAsync(): Promise<WaitingListEntry> {
  if (this.entryId === "@new") {
    this.isValid = false;
    this.entry = {
    id: "@new",
    patientId: "",
    waitingSince: new Date().toISOString(), @_important_@
    estimatedDurationMinutes: 15
    };
    this.entry.estimatedStart = (await this.assumedEntryDateAsync()).toISOString(); @_add_@
    return this.entry;
  }
}
```

In the same file, add a new function `assumedEntryDateAsync` and ``:

```tsx
private async getWaitingEntryAsync(): Promise<WaitingListEntry> {
  ...
}

private async assumedEntryDateAsync(): Promise<Date> {  @_add_@
  try {  @_add_@
    const response = await AmbulanceWaitingListApiFactory(undefined, this.apiBase)  @_add_@
      .getWaitingListEntries(this.ambulanceId)  @_add_@
    if (response.status > 299) {  @_add_@
      return new Date();  @_add_@
    }  @_add_@
    const lastPatientOut = response.data  @_add_@
      .map((_: WaitingListEntry) =>    @_add_@
          Date.parse(_.estimatedStart)   @_add_@
          + _.estimatedDurationMinutes * 60 * 1000   @_add_@
      )  @_add_@
      .reduce((acc: number, value: number) => Math.max(acc, value), 0);  @_add_@
    return new Date(Math.min(Date.now(), lastPatientOut));  @_add_@
  } catch (err: any) {  @_add_@
    return new Date();  @_add_@
  }  @_add_@
}  @_add_@
```

Add the display of the assumed entry time:

```tsx
  render() {
    return (
      <Host>
        ...
        <md-filled-text-field disabled
                        label="Čakáte od" 
                        value={new Date(this.entry?.waitingSince || Date.now()).toLocaleTimeString()}> @_important_@
                        <md-icon slot="leading-icon">watch_later</md-icon>
        </md-filled-text-field>
        <md-filled-text-field disabled @_add_@
                        label="Predpokladaný čas vyšetrenia"  @_add_@
                        value={new Date(this.entry?.estimatedStart || Date.now()).toLocaleTimeString()}> @_add_@
                        <md-icon slot="leading-icon">login</md-icon>  @_add_@
        </md-filled-text-field> @_add_@
        ...
  }
```

Save the file, and if you haven't started the development server, start it with the command `npm run start` and go to the page [http://localhost:3333/ambulance-wl](http://localhost:3333/ambulance-wl). Press the '+' button. On the screen, you will see the modified editor.

2. In the file `${WAC_ROOT}/ambulance-ufe/api/ambulance-wl.openapi.yaml`, we provided future times in the `WaitingListEntriesExample` example. We expect the assumed entry time to be at 11:15 Coordinated Universal Time (UTC). After accounting for the time zone, it should be `12:15`. Notice that the assumed entry time is (almost) the same as the patient's arrival time.

You probably already know what the problem is, but now we will show you an approach to uncovering the problem using debugging tools.

In the browser, press the _F12_ key, or open the _More tools -> Developer tools_ item in the menu (the current name and keyboard shortcut may vary between browsers; here, we provide the procedure for [Google Chrome](https://www.google.com/chrome/)). Open the `Sources` tab and in the navigation panel, open `localhost:3333/build/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.tsx`. Find the `assumedEntryDateAsync` function, and by clicking on the number of the second line of the function, set a breakpoint. By default, we assume that the calculation of the entry time must be incorrect since the displayed value does not correspond to reality.

3. Reload the page by pressing the _F5_ key. The browser will stop at the breakpoint we set in the previous step, and information about the state of the calculation at this point will be displayed in the right panel. Step through the program by pressing the _F10_ key until the last line of the `try` block in this function. Notice how the values in the right panel change in the _Scope_ tab. Add a new watch expression in the right panel - press the '+' icon in the _Watch_ section and enter the expression `new Date(Date.now()).toISOString()` and then the expression `new Date(lastPatientOut).toISOString()`. These expressions represent the values at the time when the program stopped at the breakpoint. The result should look like this:

![Setting a breakpoint](./img/120-01-Debugging.png)

Compare the times in the provided expressions - the time in the `lastPatientOut` variable corresponds to our expectation, so the error must be elsewhere. After examining the expression `Math.min(Date.Now(), lastPatientOut)`, we find that we mistakenly used the `min` operator instead of the `max` operator.

Explore other debugging interface elements. At the top of the tool, there are controls for stepping through the program. In the window, you can view the values of individual variables visible in the scope of the program breakpoint. We also see the call stack, which allows us to evaluate variable values in the scope above the current scope.

4. Fix the error in the file `${WAC_ROOT}/ambulance-ufe/src/components/<pfx>-ambulance-wl-editor/<pfx>-ambulance-wl-editor.tsx` - replace the `min` operator with the `max` operator and reload the page. Remove the breakpoint from the program and verify that the calculated value matches our expectation.

5. Debugging tools work in an environment where both compiled and source code are available. This condition is not met in a production setting. To detect the cause of potential errors or to reproduce a situation that led to an error in production, it is common to add execution recording code - logging. We will show you how to print the path that the `<pfx>-ambulance-wl-app` component is trying to handle. Open the file `${WAC_ROOT}/ambulance-ufe\src\components\<pfx>-ambulance-wl-app\<pfx>-ambulance-wl-app.tsx` and modify the `render()` function.


```tsx
render() {
  console.debug("<pfx>-ambulance-wl-app.render() - path: %s", this.relativePath);
  ...
```
  
Save the file. Go to the browser, choose the _Console_ tab in the Developer Tools, and in the drop-down list of default log levels, select the option _Details_ - _Debug_. Reload the page. In the console output, you will see the output of the `console.debug()` function.

![Console log output](./img/120-02-ConsoleLog.png)

We also have functions like `console.log()`, `console.info()`, `console.warn()`, and `console.error()`, or we can use one of the many libraries dedicated to creating execution logs. In any case, these methods should be used appropriately - too frequent calls to these methods can unnecessarily clutter the console output, using logs without corresponding information about the program's state can be useless for understanding the causes of failure, and sending logs to the server can unnecessarily burden the user's available bandwidth.

6. Archive your changes to the remote repository.

```ps
git add .
git commit -m "Predpokladaný čas vstupu"
git push
```
