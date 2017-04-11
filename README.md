# HTML As A Service (HaaS)
Imagine widget X performs some ajax operation. Ideally, widget X should update itself to properly reflect the result of that operation. Perhaps widget Y is dependent on widget X's operation. It should also be updated to reflect that operation. The *Modular Ajax Framework* provides an interface to render widget X and Y's HTML server-side to be returned (via JSON) for state replacement based on widget X's operation.  

[TOC]

## The Anatomy of `ModularAjaxCmdImpl.java`
`ModularAjaxCmdImpl.java` extends a generic class called `CostcoBaseCommandImpl.java` which provides a basic "order of operations" that maps closely to Commerce's command pattern programming model. `CostcoBaseCommandImpl.java` extends IBM's `ControllerCommandImpl.java` and executes the following methods in `performExecute`:

- **Pre processing**: logically maps to Commerce's `validateParameters` method. It's purpose is to perform any actions necessary, including validation, to the command before the command performs its primary process.
```java
protected abstract void preProcesing() throws ECException;
```
- **Run processing**: It's purpose is to determine if we should execute `process`. Run processing should return a boolean signifying whether to execute `process` or not.
```java
protected abstract Boolean runProcessing() throws ECException;
```
- **Process**: logically maps to Commerce's `performExecute` method. It's purpose is to *perform* the command's primary directive.
```java
protected abstract void process() throws ECException;
```
- **Post processing**: It's purpose is to perform any tasks *after* `process` has completed executing.
```java
protected abstract void postProcessing() throws ECException;
```
---
`ModularAjaxCmdImpl.java`'s purpose is to resolve JSON, therefore all classes extending it should forward to a view that resolves JSON. This is what `JSONLayout.jsp` is for. This JSP builds a JSON data structure used by JavaScript.

```json
{
	"metadata": ${medadata},
	"payload": {
		"html": {
			"${key}": "${escaped_html}",
			...
		},
		"data": ${data}
	},
	"error": ${error}
}
```
The most relevant method in `ModularAjaxCmdImpl.java` is `postProcessing`, which acts as the default implementation of the abstract method defined in `CostcoBaseCommandImpl`.
```java
@Override
protected void postProcessing() throws ECException {
	TypedProperty response = new TypedProperty();
	// The view being forwarded to
	response.put(ECConstants.EC_VIEWTASKNAME,
		getResponseViewName(getCommandContext()
			.getRequestProperties()));
	// This is the same as we handle errors
	// in the same response
	response.put(ECConstants.EC_ERROR_VIEWNAME,
		getResponseViewName(getCommandContext()
			.getRequestProperties()));
	// Maps to "error" in JSONLayout.jsp
	response.put(ERROR_KEY, getErrorsAsSerializedJSON());
	// Maps to "data" in JSONLayout.jsp
	response.put(DATA_KEY, getDataAsSerializedJSON());
	// Maps to "meta" in JSONLayout.jsp
	response.put(METADATA_KEY,
		getMetaDataAsSerializedJSON());
	// Controls whether or not to render JSPs
	// in JSONLayout.jsp
	response.put("hasErrors", hasErrors());
	setResponseProperties(response);
}
```
This method's responsibility is to serialize arbitrary data structures into JSON to be assigned to the keys' values in the JSON defined by `JSONLayout.jsp` above. It also calls on the *child* class's implementation of `getResponsiveViewName` to get the view name that resolves to `JSONLayout.jsp`.

## Example Implementation
The checkout process has a unique set of operations that can be conditionally switched on and off. For this reason, we've defined a class `CostcoCheckOutAjaxCmdImpl.java` that can handle executing (or not executing) these operations. This class acts as the intermediary between the extending child class and `ModularAjaxCmdImpl.java` for checkout related processes.

`SpecificOperation` -extends-> `GenericCheckoutOperation` -extends-> `ModularAjaxCmdImpl.java`

In this way we can perform a *specific* checkout operation and then dependently, based on success or failure, perform a *generic* checkout operation and finally return a JSON response to be handled by JavaScript.

`ManageGiftMessageCmdImpl.java` is an example of an augmented command that extends `CostcoCheckOutAjaxCmdImpl.java`. Its primary directive is to **save** or **delete** gift messages and then return a JSON response. The structure then is:

`ManageGiftMessageCmdImpl.java` -extends-> `CostcoCheckOutAjaxCmdImpl.java` -extends-> `ModularAjaxCmdImpl.java`.

### Set request properties
Handle setting request parameters as you normally would, setting class variables using `reqProperties`. In this case, these `reqProperties` are a result of making an AJAX request to `/ManageGiftMessageCmd` with its respective parameters.
```java
// ManageGiftMessageCmdImpl.java
@Override
public void setRequestProperties(TypedProperty reqProperties)
	throws ECException {
	super.setRequestProperties(reqProperties);

	orderId = reqProperties.getLong("orderId", null);
	addressId = reqProperties.getLong("addressId", null);
	firstName = reqProperties.getString("firstName", null);
	lastName = reqProperties.getString("lastName", null);
	fromName = reqProperties.getString("fromName", null);
	giftMessage = reqProperties.getString("giftMessage", null);
	operation = reqProperties.getString("op", null);
}
```
### Validate parameters
`CostcoCheckOutAjaxCmdImpl.java` overrides `preProcessing` and calls its own variation of `validateParameters`, `validateModularParameters`, to handle validating parameters.

```java
// CostcoCheckOutAjaxCmdImpl.java
@Override
protected void preProcesing() throws ECException {
	try {
		this.validateModularParameters();
	} catch (ECException e) {
		e.printStackTrace();
		String code = ...
		CostcoGlobalErrorMessageBean globalError =
			this.getGlobalError();
		globalError.addMessage(e.getECMessage(), code,
			e.getMessageParameters());
	}
}
```
This decorator method catches any `ECException` thrown by the extending child class and aggregates it to a `CostcoGlobalErrorMessageBean` defined in `ModularAjaxCmdImpl.java`. The extending class's implementation would look like this:

```java
// ManageGiftMessageCmdImpl.java
@Override
public void validateModularParameters() throws ECException {
	final String methodName = "validateModularParameters";

	if(firstName.length()>FIRST_NAME_LENGTH){
		throw new ECApplicationException(
			OrderExtConstants.ERR_CHECKOUT_AJAX_BAD_PARAM,
			CLASS_NAME, methodName, new String[]{...});
	}
	if(lastName.length()>LAST_NAME_LENGTH){
		throw new ECApplicationException(
			OrderExtConstants.ERR_CHECKOUT_AJAX_BAD_PARAM,
			CLASS_NAME, methodName, new String[] {...});
	}
	...
}
```
If a parameter is invalid we throw an `ECApplicationException` that gets aggregated in the `CostcoGlobalErrorMessageBean`.

### Process
Now we need to perform the primary directive of the command (saving or deleting a gift message designated by the `operation` class variable). Before `process` executes, however, `runProcessing` will determine if we meet the criteria to execute.

We define `runProcessing` in `CostcoCheckOutAjaxCmdImpl.java` to mean: "if there were no errors aggregated in `CostcoGlobalErrorMessageBean` and no exceptions aggregated in `ArrayList<Exception>`,  run the `process` method".

There should be no errors in `ArrayList<Exception>` at this point but there may have been a validation error aggregated in `CostcoGlobalErrorMessagBean`. `process` should not run in this case but `postProcessing` still needs to responsible for sending back a JSON response.

If `runProcessing` returns `true`, `process` is executed:

```java
// MangageGiftMessageCmdImpl.java
@Override
protected void process() throws ECException {
	if (OPERATION_SAVE.equals(operation)) {
		saveGiftMessage();
	} else if (OPERATION_DELETE.equals(operation)) {
		deleteGiftMessage();
	}
}
```
For the sake of explanation, we're saving a gift message for an item. If for whatever reason the **save** operation (`saveGiftMessage`) throws an error during the process, we aggregate an exception in the `ArrayList<Exception>` using the `addException` method defined in `ModularAjaxCmdImpl.java`.

### Post processing

Now that processing is over, we need to think about what we need to do as a post processing operation. At a minimum, we always want to forward to some view that resolves to `JSONLayout.jsp`. Calling `super.postProcessing()` will execute `CostcoCheckOutAjaxCmdImpl.java`'s implementation of `postProcessing`, which ultimately calls the same method defined in `ModularAjaxCmdImpl.java`, setting the proper response properties.
```java
// ManageGiftMessageCmdImpl.java
@Override
protected void postProcessing() throws ECException {
	setNeedsOrderPrepare(false);
	super.postProcessing();
}
```
Here is where those *generic* checkout operations defined in `CostcoCheckOutAjaxCmdImpl.java` can be switched on and off. By default, `CostcoCheckOutAjaxCmdImpl.java` assumes every extending class needs the `orderPrepare` operation executed. Internally `orderPepare` does a series of expensive calculations including executing `OrderCalculateCmd` and calculating shipping costs. This isn't appropriate for every single checkout related command, including `ManageGiftMessageCmdImpl.java`. Here we explicitly tell `CostcoCheckOutAjaxCmdImpl.java` to *not* execute `orderPrepare` using `setNeedsOrderPrepare(false)`.

Currently there are only two optional operations defined in `postProcessing` within `CostcoCheckOutAjaxCmdImpl.java`:

1. `orderPrepare`
2. `syncPaymentInstructionWithOrderTotal`

When implementing the Modular Ajax Framework for a command, please get clarification as to which actions require either. Additionally, any checkout specific operations that need to be defined should be aggregated in `CostcoCheckOutAjaxCmdImpl.java` and built similarly so that all extending classes have a choice to execute them or not.

NOTE: If `orderPrepare` throws an error, it adds an exception to the `ArrayList<Exception>`. If my extending child needed `syncPaymentInstructionWithOrderTotal`, it won't execute. This method has a dependency on `orderPrepare` executing successfully.

### Response view name

An extending class of `ModularAjaxCmdImpl.java` will have an implementation of `getResponseViewName`. The view name should be descriptive of the action being taken.

```java
// ManageGiftMessageCmdImpl.java
private static final String RESPONSE_VIEW = "AjaxGiftMessageUpdateView";
@Override
public String getResponseViewName(TypedProperty tp) {
	return RESPONSE_VIEW;
}
```
The view should have a corresponding struts entry:
```xml
<forward name="AjaxGiftMessageUpdateView/10851" path="giftMessageUpdate" className="com.ibm.commerce.struts.ECActionForward">
</forward>
```
### The Response

The path defined on the forward will map to a definition in `tiles-defs-json.xml` extending `JSONLayout`
```xml
<definition name="CostcoGLOBALSAS.giftMessageUpdate" extends="CostcoGLOBALSAS.JSONLayout">
	<put name="key"	value="order-items"/>
	<put name="jsp" value=".../ReviewOrderItems.jsp"/>
</definition>
```
In this example, one *key* value is defined as `order-items`. The *jsp* value corresponds with a JSP that will be rendered when `JSONLayout.jsp` is compiled. Each comma separated key is one to one with comma separated jsp values. This allows us to return N number of compiled JSPs from one response. Here's how the above definition maps to the JSON data structure rendered by `JSONLayout.jsp`:

```json
// Rendered by JSONLayout.jsp
{
	"metadata": {},
	"payload": {
		"html": {
			"order-items":
				"<div id='order-items'>
					<!-- various items -->
				</div>",
			...
		},
		"data": {}
	},
	"error": {}
}
```
Here we see the key `order-items` has the value of the rendered JSP, `ReviewOrderItems.jsp`. We can now handle updating the state of the view using this result.

### Updating state

The whole process of calling the command, retrieving a response, and updating the state of the view for the user should be encapsulated by the `ajax_update_state`
 function defined in `checkout_helper.js`.
```javascript
// orderitems.js: line ~245
...
COSTCO.Checkout.ajax_update_state({
	error_container: error_container,
	method: 'POST',
	url: '/ManageGiftMessage',
	data: data.serializeArray(),
	dataType: 'json'
})
...
```
Here we POST to our Modular Ajax Command with `/ManageGiftMessage`, sending parameters with `data`.

To protect the system and the user from unwanted actions while the system processes this request, `ajax_update_state` will disable any children DOM elements whose parent container includes the class `order-ajax`. The `always` promise resolves and reenables these elements after everything is finished.

If no `done` function is provided, `ajax_update_state` executes a default implementation, `ajax_update_default_done`, when the `done` promise resolves. The two primary functions we're interested in are `ajax_replace_content` and `ajax_handle_errors`. `ajax_replace_content` uses the keys defined in the JSON response to naively replace the DOM. For example, if my response looks like this:
```json
{
	...
	"payload": {
		"html" : {
			"order-items":
			"<div id='order-items'>
				<!-- I'm a new state! -->
			</div>"
		}
	}
}
```
We should have a corresponding HTML block that represents "order-items", with an ID attribute that matches the key:
```html
<div id="order-items">
	<!-- I'm going to be replaced! -->
</div>
```
Matching the structure of the HTML between what the JSON response returns and the HTML to be replaced is imperative for subsequent update operations to replace their states properly.

### Client-side error handling
We need to designate an error container when we execute `ajax_update_state`. The `error_container` property tells `ajax_handle_errors` where to drop any errors from the JSON response.

```javascript
COSTCO.Checkout.ajax_update_state({
	error_container: '.error-container'
	...
})
```
In the above example, `error_container` will use a DOM element(s) defined by class `.error-container` to inject errors.

`ModularAjaxCmdImpl.java` packages up any errors aggregated in `CostcoGlobalErrorMessageBean` and exposes them in the `application` property as an array. These are the results returned from throwing a validation error caused by faulty user input. The `exception` property is sourced from the `ArrayList<Exception>`, which represents *system* level and therefore generic errors.
```json
// partially omitted response
{
	"error": {
		"application": [
			"Sorry, first name was invalid",
			"Sorry, last name was invalid"
		],
		"exception": [
			"An unexpected error has occurred"
		]
	}
}
```
These exceptions end up being appended inside the `.error-container` defined when calling `ajax_update_state`. Each subsequent call erases the previous call's errors. You'll also notice by examining `JSONLayout.jsp` that the JSPs for a particular operation don't even compile when an error occurs. This saves server side compilation operations and retains the broken state that a user could potentially fix with different input.

### Gotchas

In the case where you may be augmenting an existing command, make sure that the `type` attribute of the `action` entry is `BaseAction` and not `AjaxAction`. `AjaxAction` will always reroute the response to a home grown IBM `AjaxActionResponse.jsp` which isn't the structure we want in this case.

```xml
<action path="/ManageGiftMessage" parameter="..." type="com.ibm.commerce.struts.BaseAction" />

```
