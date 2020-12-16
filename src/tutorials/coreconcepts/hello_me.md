\#S:EXTERNAL=rust=hello_me.rs
\#S:MODE=gui
\#S:EXTERNAL=html=hello_me.html=gui
\#S:EXTERNAL=javascript=hello_me_gui.js=gui
\#S:MODE=test
\#S:EXTERNAL=javascript=hello_me.js=test
# Hello Me

!!! tip "Time & Level"
    Time: ~4 hours | Level: Beginner

Welcome to another tutorial in our Core Concepts series.

Currently, the app we've built returns a constant value, but it would be useful to be able to store data for more complex applications.

This tutorial builds on the [previous](../hello_gui) tutorial, so go back and complete it if you haven't already.

### What will you learn
You'll learn how to add an entry type to your zome.
An entry is a piece of data in your source chain that's been validated.
We'll also show you how to define and validate an entry type that represents a person, and to create and read this data through zome calls.
You will also set up tests and your GUI.

### Why it matters
Storing data is at the core of Holochain.
Holochain's most important job is ensuring agents handle and store data according to the rules of your application.

## Test first

Start by writing a test to make it easy to see when your app is working:

Open up `cc_tuts/test/index.js`.

This is how we left the testing scenario in the [Hello Test](../hello_test) tutorial:

\#S:MODE=test
\#S:SKIP
```javascript
orchestrator.registerScenario('Test hello holo', async (s, t) => {
  const {alice, bob} = await s.players({alice: config, bob: config}, true);
  const result = await alice.call('cc_tuts', 'hello', 'hello_holo', {});
  t.ok(result.Ok);
  t.deepEqual(result, {Ok: 'Hello Holo'});
```
The new tests go below `t.deepEqual(result, { Ok: 'Hello Holo' })`
The following test will create an entry with the name ‘Alice’, retrieve the same entry, and check that it has the name ‘Alice’.

Add a call to the `create_person` function with a person named Alice:
\#S:INCLUDE
```javascript
  const create_result = await alice.call('cc_tuts', 'hello', 'create_person', {
    person: {name: 'Alice'},
  });
```

Check that the result of the call is Ok:

```javascript
  t.ok(create_result.Ok);
  const alice_person_address = create_result.Ok;
```

Tell the test to wait for the DHT to become consistent.

```javascript

  await new Promise(r => setTimeout(r, 1000));

```

!!! warning "Shouldn't I use `s.consistency()`?"
    If this isn't the first time you've written a Tryorama scenario, you may remember the statement `await s.consistency()` which waited for the DHT to become consistent. We're not using this because, in the course of fixing another bug, the `s.consistency()` function was broken. We made the decision to not fix it now, but instead introduce better tools for consistency checking later. We're currently working on [a new Holochain](https://redux.developer.holochain.org/holochain-rsm-guidance.html) and will not be backporting this fix to the soon-to-be-deprecated version that you're currently learning with.

Add a call to the `retrieve_person` function with the address from the last call:

```javascript
  const retrieve_result = await alice.call(
    'cc_tuts',
    'hello',
    'retrieve_person',
    {address: alice_person_address},
  );
```

Check that this call is Ok as well:

```javascript
  t.ok(retrieve_result.Ok);
```
This is the actual result we want at the end of the test. Check that the entry at the address is indeed named `Alice`:

```javascript
  t.deepEqual(retrieve_result, {Ok: {name: 'Alice'}});
```
\#S:HIDE
```javascript
});

orchestrator.run();
```

### Run sim2h
Again, you will need to run the sim2h server in a separate terminal window:

!!! note "Run in nix-shell"
    ```bash
    sim2h_server
    ```

### Running the test
Your test should now look like this:

\#S:CHECK=javascript=test

Obviously, right now, these tests will fail. Can you guess what the first failure will be? Let's have a look.

Enter the nix-shell if you don't already have it open:

```bash
nix-shell https://github.com/holochain/holonix/archive/release-0.0.85.tar.gz
```

Run the test:

!!! note "Run in nix-shell"
    ```bash
    hc test
    ```

!!! failure "The test fails on the create_person function because it doesn't exist yet:"
    `"Holochain Instance Error: Zome function 'create_person' not found in Zome 'hello'"`

> Note: This test might actually get stuck because we haven't put in the required functions yet. Press `ctrl-c` to exit a stuck test.

## Add the entry

Open up your `zomes/hello/code/src/lib.rs` file.
To add an entry to your source chain, begin by telling Holochain what kind of entry exists.
First, we'll create a [`struct`](https://doc.rust-lang.org/1.9.0/book/structs.html) to define the shape of the data.

We will add a `Person` struct in a moment, but this is where to put it:

\#S:SKIP
```rust
// <---- Add the person struct here.

#[zome]
mod hello_zome {
```

Add the following lines:

Allow this struct to be easily converted to and from JSON:

\#S:INCLUDE
```rust
#[derive(Serialize, Deserialize, Debug, DefaultJson, Clone)]
```

Represent a person as a struct:

```rust
pub struct Person {
```

Represent their name as a string:

```rust
    name: String,
}
```

Look for the following lines inside the `hello_zome` mod:

\#S:SKIP
```rust
#[zome]
mod hello_zome {
```

\#S:INCLUDE
\#S:EXTERNAL=rust=hello_me_2.rs

\#S:SKIP
```rust
  /* --- Lines omitted -- */
  #[zome_fn("hc_public")]
  pub fn hello_holo() -> ZomeApiResult<String> {
      Ok("Hello Holo".into())
  }

  // <---- Add the following lines here.
```

Add the `person_entry_def` function, which tells Holochain about the person entry type:

\#S:INCLUDE,HIDE
```rust

```

```rust
    #[entry_def]
    fn person_entry_def() -> ValidatingEntryType {
```

Add the `entry!` macro, which lets you easily create a `ValidatingEntryType`:

```rust
        entry!(
```

To be consistent, give it the same name as the `Person` struct. Entry types are usually in lowercase.

Add the name and description of the entry:

```rust
            name: "person",
            description: "Person to say hello to",
```

Entries of this type are just for this agent's eyes only, so set the entry sharing to private:

```rust
            sharing: Sharing::Private,
```

Add the `validation_package` function that says what is needed to validate this entry:

```rust
            validation_package: || {
                hdk::ValidationPackageDefinition::Entry
            },
```

Add the `validation` function that validates this entry.

As long as it fits the shape of the `Person` struct, it returns that this entry is always Ok:

```rust
            validation: | _validation_data: hdk::EntryValidationData<Person>| {
                Ok(())
            }
        )
    }
```

Now, you can create actual `person` entries and store them on your source chain.

> __Note:__
> Validation is very important. It's the "rules of the game" for your Holochain app. It is meaningful to emphasize that although we are returning `Ok(())`, we're still validating that the data type checks as a `Person` with a `name` property containing a `String`. Essentially, this rule says that the person entry must be in this format.

\#S:EXTRA
```rust
    }
```

\#S:CHECK=rust

## Create a person

You now need a way for your UI to actually create a person entry. Holochain has a concept called `hc_public`, which is a way of telling the runtime to make this function available to call from outside the zome.

Add the following lines below the previous `person_entry_def` function:

Add a public function that takes a `Person` and returns a result with an `Address`:

\#S:HIDE,INCLUDE
```rust

```

\#S:INCLUDE
```rust
    #[zome_fn("hc_public")]
    pub fn create_person(person: Person) -> ZomeApiResult<Address> {
```

Create an entry from the person argument:

```rust
        let entry = Entry::App("person".into(), person.into());
```

Commit the entry to your local source chain:

```rust
        let address = hdk::commit_entry(&entry)?;
```

Return the `Ok` result with the new person entry's address:

```rust
        Ok(address)
    }
```

### Compile

\#S:EXTRA
```
    }
```

\#S:CHECK=rust

Check for compile errors again:

!!! note "Run in nix-shell"
    ```bash
    hc package
    ```

## Retrieve person

Lastly, you need a way for your UI to get a person entry back from the source chain.

Add the following lines below the `create_person` function:

Add a public `retrieve_person` function that takes an `Address` and returns a `Person`:

\#S:HIDE,INCLUDE
```

```

```rust
    #[zome_fn("hc_public")]
    pub fn retrieve_person(address: Address) -> ZomeApiResult<Person> {
```

Get the entry from your local storage, asking for it by address, and convert it to a Person type:

```rust
        hdk::utils::get_as_type(address)
    }
```

\#S:HIDE,INCLUDE
```rust
}
```

> In Rust, the last line is always returned---you do not need to explicitly say `return`---just leave off the `;`.

### Test

\#S:CHECK=rust

Instead of directly compiling, you can run the test you wrote at the start (the test always compiles before it runs):

!!! note "Run in nix-shell"
    ```bash
    hc test
    ```

!!! success "If everything went smoothly you will see:"
    ```bash
    # tests 5
    # pass  5

    # ok
    ```

## UI

Now that the back end is working, you can modify the UI to interact with zome functions you created.

Go to the GUI project folder you created in the [Hello GUI](../hello_gui) tutorial:

```bash
cd holochain/coreconcepts/gui
```

\#S:MODE=gui
\#S:INCLUDE

Start by adding the HTML elements to create a person in your `index.html`.

Look for the previous 'say hello' elements.

\#S:SKIP

```html
    <button onclick="hello()" type="button">Say Hello</button>
    <div>Response:</span><span id="output"></div>
    <!-- Put the following lines here -->
```

\#S:INCLUDE
Below them, give the section a heading:
```html
    <h3>Create a person</h3>
```

Add a text box so the user can enter their name:

```html
    <input type="text" id="name" placeholder="Enter your name :)">
```

Add a button that calls a (yet to be written) JavaScript function called `create_person`:

```html
    <button onclick="create_person()" type="button">Submit Name</button>
```

Add a span with the id `address_output` so you can render the result of this call:

```html
    <div>Address: <span id="address_output"></span></div>
```

\#S:EXTRA
```
        <script
          type="text/javascript"
          src="hc-web-client-0.5.3/hc-web-client-0.5.3.browser.min.js"
        ></script>
        <script type="text/javascript" src="hello.js"></script>
      </body>
    </html>
```

\#S:CHECK=html=gui

### Switch to your `hello.js` file


Let's write the `create_person` function that will call your zome.

Add the `create_person` function:

```javascript
function create_person() {
```

Get the text box by its ID `name` and save the current text value into the name variable:

```javascript
  const name = document.getElementById('name').value;
```

Wait for the connection; then make a zome call:

```javascript
  holochain_connection.then(({callZome, close}) => {
```

Call `create_person` in your `hello` zome and pass in the name variable as part of a person structure, then write the result to the console:

```javascript
    callZome('test-instance', 'hello', 'create_person')({
      person: {name: name},
    }).then(result => console.log(result));
  });
}
```

\#S:CHECK=javascript=gui

### Run the server and open a browser

Let's test your first call.

Open a new terminal window and enter the nix-shell:

!!! note "Run in nix-shell"
    ```bash
    hc package
    ```

    Copy the new DNA hash into your `bundle.toml` file; you can use the [update script from the previous tutorial](../hello_gui/#run-the-bundle). Then start up your hApp:

    ```bash
    hc run
    ```

    Your DNA hash will have changed, so you'll likely get the hash mismatch error again. [Follow the instructions from the last tutorial](../hello_gui/#run-the-bundle) to get it working.

Now that both your UI server and your Holochain conductor server are running, open up a browser and go to `127.0.0.1:8888`. You should see the HTML elements you created:

![](../../img/create_person_1.png)

Open the developer console, enter your name, and press the "Submit Name" button. You should see something similar to this:

![](https://i.imgur.com/s20Oh6A.png)
> The address you see will probably be different, because you typed in your own name.

### Show the new entry's address

We're going to show the address on the page now, rather than the developer console.

First, a bit of refactoring---if you make the `show_ouput` function more generic, you can reuse it for each element that shows the output for a zome function.

Pass in the element's ID so that the function can be reused:

\#S:CHANGE
```diff
-function show_output(result) {
+function show_output(result, id) {
-  var span = document.getElementById('output');
+  var el = document.getElementById(id);
  var output = JSON.parse(result);
```
It would also be nice to have some error checking.
Add in a if statement that checks that `Ok` is not null:
\#S:CHANGE
```diff
+  if (output.Ok) {
-  span.textContent = output.Ok;
+    el.textContent = output.Ok;
+  } else {
+    alert(output.Err.Internal);
+  }
}
```
\#S:CHANGE
```diff
function hello() {
  holochain_connection.then(({callZome, close}) => {
    callZome(
      'test-instance',
      'hello',
      'hello_holo',
-    )({args: {}}).then(result => show_output(result));
+    )({args: {}}).then(result => show_output(result, 'output'));
  });
}
```
\#S:CHANGE
```diff
function create_person() {
  const name = document.getElementById('name').value;
  holochain_connection.then(({callZome, close}) => {
    callZome('test-instance', 'hello', 'create_person')({
      person: {name: name},
-    }).then(result => console.log(result));
+    }).then(result => show_output(result, 'address_output'));
  });
}
```

\#S:CHECK=javascript=gui

### Enter the browser

Go back to your browser and refresh the page. This time, when you enter your name and press __Submit Name__, you will see the address show up:

![](../../img/create_person_2.png)

## Retrieve a person entry and show it in the UI

In the `index.html` file, under the create person section, add a new header:

```html
    <h3>Retrieve Person</h3>
```

Add a text box so the user can enter the address that is returned from the `create_person` function:

```html
    <input type="text" id="address_in" placeholder="Enter the entry address">
```

Add a button that calls the (yet to be written) `retrieve_person` JavaScript function:

```html
    <button onclick="retrieve_person()" type="button">Get Person</button>
```

Add a span with the ID `person_output` to display the person that is returned from the `retrieve_person` function:

```html
    <div>Person: <span id="person_output"></span></div>
```

\#S:HIDE
```html
    <script
      type="text/javascript"
      src="hc-web-client-0.5.3/hc-web-client-0.5.3.browser.min.js"
    ></script>
    <script type="text/javascript" src="hello.js"></script>
  </body>
</html>
```

\#S:CHECK=html=gui


### Go to your `hello.js` file

Add the `retrieve_person` function to call the zome function of the same name and show the response:

```javascript
function retrieve_person() {
```

Get the value from the `address_in` text box:

```javascript
  var address = document.getElementById('address_in').value.trim();
```

Wait for the connection; then make a zome call:

```javascript
  holochain_connection.then(({callZome, close}) => {
```

Call the `retrieve_person` public zome function, passing in the address. Then pass the result to `show_person`:

```javascript
    callZome('test-instance', 'hello', 'retrieve_person')({
      address: address,
    }).then(result => show_person(result, 'person_output'));
  });
}
```

Add the `show_person` function. It is very similar to `show_output`, except that you need to show the name.

```javascript
function show_person(result) {
  var person = document.getElementById('person_output');
  var output = JSON.parse(result);
  if (output.Ok) {
    person.textContent = output.Ok.name;
  } else {
    alert(output.Err.Internal);
  }
}
```

\#S:CHECK=javascript=gui

### Enter the browser
Finally, go and test this at `127.0.0.1:8888`.
You should see something like this:
![retrieving a person](../../img/create_person_3.png)

Well done! You have stored and retrieved data from a private source chain using a GUI!

!!! success "Solution"
    You can check the full solution to this tutorial on [here](https://github.com/freesig/cc_tuts/tree/hello_me).

## Key takeaways
- Entries can be defined using Rust types.
- Entry definitions tell Holochain about the data it can hold and how to validate it.
- Once an entry is committed this can never be undone, and any other agent running the same DNA will always commit an entry that has passed validation. _(This means validation must be deterministic.)_
- The zome returns entries in the JSON format.

## Learn more
- [JSON](http://www.json.org/)
- [Struct](https://doc.rust-lang.org/book/ch05-00-structs.html)
- [CALM](http://bloom-lang.net/calm/)
