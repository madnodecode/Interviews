# A brief guide to becoming productive with FilterCoffee

This guide is aimed to help people who plan on using the new functional testing rails (FilterCoffee) in the near future. We will start by covering the motivation behind building FilterCoffee, its architecture, and what features it provides to simplify the testing process. We will then get hands-on with some real examples that should help you understand how the test are written for different use-cases. Finally, we illustrate what the introduction of a new service (REST/ASF) typically entails. By the end of this guide you should be equipped to add tests for existing and new features/products. If you have any questions feel free to email me at sahjain@paypal.com

## Table of Contents
* **[Setup](#setup)**
* **[Motivation](#motivation)**
* **[Architecture](#architecture)**
 * **[Arriving at the design](#arriving-at-the-design)**
 * **[An overview of atomic units: ApiUnit, ValidationUnit, ExecutionUnit](#an-overview-of-atomic-units-apiunit-validationunit-executionunit)**
 * **[A quick tour under the hood](#a-quick-tour-under-the-hood)**
 * **[Architectural diagram](#architectural-diagram)**
* **[Examples](#examples)**
 * **[Writing an end to end test](#writing-an-end-to-end-test)**
 * **[Adding a new payload template](#adding-a-new-payload-template)**
 * **[Adding a new field validation](#adding-a-new-field-validation)**
 * **[Testing your WOWO based feature](#testing-your-wowo-based-feature)**
 * **[Validating the Mayfly session](#validating-the-mayfly-session)** 
 * **[Introducing a new endpoint](#introducing-a-new-endpoint)**
 * **[Analyzing different flavors of tests](#analyzing-different-flavors-of-tests)**
 * **[Dynamic Payloads and Dynamic Validations](#dynamic-payloads-and-dynamic-validations)**
* **[Tagging your tests](#tagging-your-tests)**


##Testing against dev host
When running functional tests against local dev host, point symphony to msmaster.qa.paypal.com (or any other dependent stage) and use following VM arguments in your functional test

```
-DJAWS_HOSTNAME=msmaster.qa.paypal.com
-DUSE_LOCAL_SYMPHONY=true
-DSYMPHONY_PORT=8080
-DJAWS_SSH_RESTRICTED_STAGE=true
``` 

If you running your functional test against a particular userstage, say functionaltest.qa.paypal.com, you will have to add SYMPHONY_HOST=functionaltest.qa.paypal.com and SYMPHONY_PORT=17266. 17266 is the port on which paymentapiplatserv runs.
```
-DJAWS_HOSTNAME=msmaster.qa.paypal.com
-DUSE_LOCAL_SYMPHONY=true
-DSYMPHONY_HOST=functionaltest.qa.paypal.com
-DSYMPHONY_PORT=17266
-DJAWS_SSH_RESTRICTED_STAGE=true
```

```
Functional Test by default do 1 Retry if any Rest API failed with 500(Internal Server Error) if you wish to do multiple retry you can add following JVM Argument. Each retry count will add 20sec delay to previous delay.
For an Ex. If i have -DRETRY_ERROR=2
Let's say first try got failed.Then first retry will happen after 20sec after first try was failed and complete.Second Retry happen after 40sec of First Retry.
So it will do total 2 retry for given api failed with 500.
```
-DRETRY_ERROR=3
```

In addition, if you have your JAVA_HOME set to 1.8 or have multiple version of JAVA installed, you may also want to add additional parameters specifying source and target. In this case, add the following along with above mentioned parameters:-
```
-Dsource=1.7 -Dtarget=1.7 
```
This will avoid any Java cross-compilation issue which might arise. This issue is well explained in the thread [here](https://gist.github.com/AlainODea/1375759b8720a3f9f094)

##Setup
* Import the project
 * Make sure you've cloned the [symphony code](https://github.paypal.com/Payments-R/paymentapiplatserv.git) and imported all the maven projects into your IDE via the parent pom file.
 * Verify that your workspace has a maven project called "functional-tests". If for some reason you don't find it, click on File -> Import -> Existing Maven Projects -> {{Browse folder where the code is downloaded}} -> Select functional-tests -> Finish
* Install TestNG plugin
 * TestNG plugin from the Eclipse Marketplace or their regular update site brings in version 6.9.10 which is a version incompatible with the version paymentapiplatserv currently uses, version 6.9.13.
 * For Installing TestNG version 6.9.13, click on Click on Help -> Install New Software -> {{Type out "http://beust.com/eclipse-old/eclipse_6.9.13.201609291640/" in the search bar, without any quotes}} -> Install.
* Configure VM arguments to access stage
 * If you've authored a test (or located one that you're interested in running), here's a general pattern: Click on Run -> Run Configurations -> TestNG -> {{Browse Project and select functional-tests}} -> {{Browse class and/or method}} -> Apply. Once you're done with this, click on Arguments (tab) -> {{ In the VMArguments text box, type out your desired stage name prefixed by -DJAWS_HOSTNAME}}. So if you're running a test method against stage2XXXX.qa.paypal.com, the VMArguments should look like -DJAWS_HOSTNAME=stage2XXXX.qa.paypal.com
* Setup passwordless access
  * This is required to enable lower-level libraries to SSH into the staging environment freely. If you are interested in running a test against a stage (like stage2XXXX) then in addition to tweaking the VMArguments, you need to set up passwordless access to that particular stage. [Documentation on this can be found here](https://confluence.paypal.com/cnfl/display/QI/Passwordless+Application).
* {{OPTIONAL}} Run functional tests against "local" server
 * If you are interested in debugging a particular test case while it runs against your local symphony server, follow the steps below:
   * Run the symphony server against a particular stage in DEBUG mode
   * Ensure the filterCoffee test's VM arguments also point to that same stage
   * Add the following VM arguments to the filterCoffee test: ``` -DUSE_LOCAL_SYMPHONY=true ```
   * This ensures that every api invocation that "can" be handled by your local symphony server will be routed there. All other api calls (ex: SetEC, DoEC, Wallet.ADD) get routed to the relevant service on the stage (and not locally).
  * **Note**: _If see errors that look like this when you run your Functional Tests, it means that your ssh setup needs fixin'_ 
 
      ```
       04.15.2015 11:26:10.892,[1] com.paypal.test.jaws SEVERE Packet corrupt
      com.jcraft.jsch.JSchException: Packet corrupt
      at com.jcraft.jsch.Session.start_discard(Session.java:1049)
      at com.jcraft.jsch.Session.read(Session.java:919)
      at com.jcraft.jsch.Session.connect(Session.java:309)
      at com.jcraft.jsch.Session.connect(Session.java:183)
      at com.paypal.test.jaws.execution.ssh.impl.JSchClientImpl.getSession(JSchClientImpl.java:398)
      at com.paypal.test.jaws.execution.ssh.impl.JSchClientImpl.executeCommand(JSchClientImpl.java:321)
       ```
  * Follow these steps and you should be golden.
    1. ```mv ~/.ssh ~/.ssh.copy```
    2. Follow the instructions @ https://help.github.com/enterprise/2.1/user/articles/generating-ssh-keys/ to ensure your github access continues. Make sure you dont use a passphrase to set your RSA key-pair up!
    3. Your ~/.ssh should have been re-created now
    4. Setup trust to your test stages and other hosts you had setup previously using `passwordless <hostname>`

##Motivation
FilterCoffee started as an attempt to simplify the functional testing process and improve the reliability of our CI. With several lessons learned from the existing functional tests, we sought to address the following: 
* Wide array of redundancies across test cases and packages
* Poor discoverability of potentially re-usable code
* Inconsistencies in testing quality/validations
* Verbosity of test cases and general util/base classes
* Unnatural design for writing data-driven tests
* Slow performance in execution time
* Lack of reliability in the face of parallel execution
* Challenges in maintainability

The existing functional tests' biggest strength was its flexibility, but that turned out to be a double edged sword which in turn led to many of the problems outlined above. It became apparent that the right way to address the above problems was to introduce a set of patterns; these patterns together is what we've dubbed a "framework/rails" of the name FilterCoffee.

##Architecture:
This section covers the general architectural design in FilterCoffee. It starts by illustrating the simple thought process behind building the main FilterCoffee components and surveys the role they play. After providing a sense of the general flow of a test case in the FilterCoffee rails, it outlines a basic sequence diagram depicting interactions between between building blocks.

###Arriving at the design
Instead of starting with a daunting UML diagram, we'll walk through a basic functional test iteratively and see how it could motivate the design for better rails like FilterCoffee. Please note that this is simply a discussion of design requirements at a conceptual level. If you feel like you already understand filterCoffee reasonably well, free to skip this section.

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  SetEC();
  createCheckoutSession();
  approveCheckoutSession();
  GetEC();
  DoEC();
}
```

The above is a highly stripped down version of what a simple end to end orchestration may look like, but there are several key missing pieces. For starters:
* Methods like SetEC() and DoEC() need to operate on the seller's information. createCheckoutSession() and approveCheckoutSession() need both the buyer and seller. How can these methods not consume this critical information?

This is a legitimate concern, so let's go ahead and add the relevant inputs:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  SetEC(seller);
  createCheckoutSession(buyer, seller);
  approveCheckoutSession(buyer, seller);
  GetEC(seller);
  DoEC(seller);
}
```

While this is a mild improvement, it raises more important questions:
* SetEC() and DoEC() on their own don't make sense. After all, there could be several flavors of SetEC/DoEC. How can we dictate which one we want to use?

There's many ways to tackle this. You could manually take up the responsibility of creating the request on your own and supply the request as a parameter to the SetEC() method. This would look a little like:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  NVPPostImpl ecRequest = buildSetECRequest();
  SetEC(seller, ecRequest);
  createCheckoutSession(buyer, seller);
  approveCheckoutSession(buyer, seller);
  GetEC(seller);
  DoEC(seller);
}

public NVPPostImpl buildSetECRequest() {
  NVPPostImpl ecRequest = new NVPPostImpl(merchant.nvp.url);
  ecRequest.addParam("METHOD", "SetExpressCheckout");
  ecRequest.addParam("VERSION", "82");
  ecRequest.addParam("USER", seller.apiCredentials().getApiUserName());
  ecRequest.addParam("PWD", seller.apiCredentials().getApiPassword());
  ecRequest.addParam("SIGNATURE", seller.apiCredentials().getSignature());
  ecRequest.addParam("AMT", "5");
  ecRequest.addParam("RETURNURL", "http://xotools.ca1.paypal.com/ectest/return.html");
  ecRequest.addParam("CANCELURL", "http://xotools.ca1.paypal.com/ectest/cancel.html");
  ecRequest.addParam("CURRENCYCODE", "USD");
  ecRequest.addParam("PAYMENTACTION", "SALE");
  ecRequest.addParam("LANDINGPAGE", "Billing");
  return ecRequest
}
```

The issue with the above approach is the fact that once you want to incorporate 20 different flavors of SetEC, it begins polluting your test code and becomes hard for tests in other classes to locate and use these methods. So it's clear this needs to reside in some sort of a central location so that it's easy for other tests to be able to tap into it, but is the construction of the request from the test code necessarily the best approach? Naturally, there is no escaping the work of creating the request, so it really becomes a question of where and how the construction happens.

Let's try another approach. Perhaps we could try to "label" the flavor of request that we want to create by a simple alias of some sort. If you review the request construction code above, it's clear we're building a very basic SetEC request for a single item for a SALE (and not auth/order) in USD. So maybe label an alias for this request as SIMPLE_SALE_USD? A simple java construct to hold such simple "labels" is an enum, so we could in theory define a SetEC request-based enum (let's call it SetECTemplate) which holds all the various SetEC request labels. Make no mistake, so far all we've done is found a cosmetically favorable alternative by succinctly labelling a request with an enum value, but the actual work of constructing the request still needs to be done. In order to complete this idea, it's important to discuss the idea behind what the SetEC() method actually does.

Visualize SetEC() as a separate method that simply "handles" SetEC interactions. This means that it would be responsible for doing 3 things:
 * Build a request based on some label passed by the test code
 * Call the api with the request body and headers
 * Get the response and validate it

This ends up looking a little like:

```java
class MyTests {

  public void endToEndTestPseudoCode() {
    // orchestration assuming buyer/seller have been created
    
    // newer code
    SetECHandler setEcHandler = new SetECHandler();
    setEcHandler.setEC(seller, SetECTemplate.SIMPLE_SALE_USD);
    ...
  }
}

class SetECHandler {

  public setEC(PayPalUser seller, SetECTemplate template) {
    // map the request
    NVPPostImpl ecRequest = SetECRequestMapper.map(template);
    // call the api
    Response response = callApi(request);
    // validate the response
    validateResponse(response);
  }
}

class SetECRequestMapper {
  public map(SetECTemplate template) {
    switch(template){
      case SIMPLE_SALE_USD:
        return buildSimpleSetECRequest();
      default: break;
    }
  }
  
  private NVPPostImpl buildSimpleSetECRequest() {
    NVPPostImpl ecRequest = new NVPPostImpl(merchant.nvp.url);
    ecRequest.addParam("METHOD", "SetExpressCheckout");
    ecRequest.addParam("VERSION", "82");
    ecRequest.addParam("USER", seller.apiCredentials().getApiUserName());
    ecRequest.addParam("PWD", seller.apiCredentials().getApiPassword());
    ecRequest.addParam("SIGNATURE", seller.apiCredentials().getSignature());
    ecRequest.addParam("AMT", "5");
    ecRequest.addParam("RETURNURL", "http://xotools.ca1.paypal.com/ectest/return.html");
    ecRequest.addParam("CANCELURL", "http://xotools.ca1.paypal.com/ectest/cancel.html");
    ecRequest.addParam("CURRENCYCODE", "USD");
    ecRequest.addParam("PAYMENTACTION", "SALE");
    ecRequest.addParam("LANDINGPAGE", "Billing");
    return ecRequest
  }
}
```

This feels like a cleaner design because there's a clearer separation of concerns. You can now pass the SetECTemplate from the test code without a sweat and be sure that other people can use it too. If you think about it further, it might make more sense to "collect" all the expressCheckout related functionality under the same class. That is, maybe we could just have a single class called ExpressCheckoutHandler and have public methods of SetEC, GetEC and DoEC inside which in turn would delegate request mapping, call the apis, validate responses. In addition to this we could also collect the request mapping under a common class called ExpressCheckoutRequestMapper (which would have separate methods like mapSetECRequest() and mapDoECRequest()). And why stop at refactoring just EC-related apis, let's extend this design to all CheckoutSession-related apis. If we do this, our test code might look a little like:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  
  // assume all handlers have been instantiated
  ECHandler.setEC(seller, SetECTemplate.SIMPLE_SALE_USD);
  XOSessionHandler.createCheckoutSession(buyer, seller);
  XOSessionHandler.approveCheckoutSession(buyer, seller);
  ECHandler.getEC(seller);
  ECHandler.doEC(seller, DoECTemplate.SOME_DOEC_TEMPLATE);
}
```

So let's just recap what you've done so far. In our test, you've defined what api you wanted to call (ex: SetEC), what entities you want involved (buyer/seller), and you've also passed the label that dictates how to build the request (ex: SetECTemplate.SIMPLE_SALE_USD). The label/enum/alias is nothing more than a template that influences the construction of the payload (usually a request body) in the requestMapper, so maybe we'll call it a payloadTemplate instead. We authored some additional classes to handle different concerns (ExpressCheckoutHandler, ExpressCheckoutRequestMapper) and we can feel better about the design because it's more maintainable.

This alleviates some anxiety, but wait! We've arrived at yet another important question:
* createCheckouSession() couldn't possibly be successful without providing the cart_id in the request body/payload. The cart_id comes from the SetEC response, so who's taking care of that? approveCheckoutSession() also can't be successful without providing the cart_id in the uri itself, so who's taking care of that?

Before we fret about how solve the whole problem, let's think about the question. The gap is in being able to construct a request body for the createCheckoutSession api. Staying true to our design, this means the responsibility of constructing an appopriate request lies squarely in the CheckoutSessionRequestMapper. So we need to somehow communicate the SetEC response all the way in the test code to the CheckoutSessionRequestMapper. There's a couple of ways to do this; let's start with a simple approach:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  
  // assume all handlers have been instantiated
  Response setECResponse = ECHandler.setEC(seller, SetECTemplate.SIMPLE_SALE_USD);
  XOSessionHandler.createCheckoutSession(buyer, seller, setECResponse);
  ...
}
```

This is pretty straightforward, but let's think a little more about it. If we do this, then what we're saying is that in order to reach the relevant requestMapper, the intermediate handler method needs to have the right interface to accept the right data. This means that if we wanted to really use this pattern for our test, the correct way to write the test would be:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  
  // assume all handlers have been instantiated
  SetECResponse = ECHandler.setEC(seller, SetECTemplate.SIMPLE_SALE_USD);
  XOSessionHandler.createCheckoutSession(buyer, seller, SetECResponse);
  XOSessionHandler.approveCheckoutSession(buyer, seller, SetECResponse);
  GetECResponse = ECHandler.getEC(seller, setECResponse);
  ECHandler.doEC(seller, DoECTemplate.SOME_DOEC_TEMPLATE, SetECResponse, GetECResponse);
}
```
This works fine for this particular use-case, but it's not hard to think of some use-cases that don't have such a standard interface. For example, while building a request payload for creating a Payment, sometimes we need a fundingId from a preceding GFO call, sometimes a payerId, sometimes both, and sometimes none of them but a full json filePath instead. You could create a separate createPayment method for each use-case, but then we give in to the curse of polymorphic pollution. We could create a method with placeholders for all arguments, but then you risk many invocations that look like createPayment(1234, null, null, null) and a pollution of your test code with many permutations of method invocations. Could we do better?

Perhaps we could continually supply the preceding api results via a local transaction context of some sort that holds all the responses of each api executed. In other words, this wouldn't be much more than a simple map that captures each api's result and is passed around. This might look a little like:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  
  // create map that is responsible for "holding" results
  OrchestrationContext orchestrationContext = new OrchestrationContext();
  
  // call api and store result
  SetECResponse = ECHandler.setEC(SetECTemplate.SIMPLE_SALE_USD, seller, orchestrationContext); // empty map at this point
  orchestrationContext.storeResponse(SetEC, SetECResult);
  
  // call api and store result
  CreateXOSessionResponse = XOSessionHandler.createCheckoutSession(buyer, seller, orchestrationContext); // orchestrationContext has SetEC result
  orchestrationContext.storeResponse(CreateCheckoutSession, CreateXOSessionResponse);
  
  // call api and store result
  ApproveXOSessionResponse = XOSessionHandler.approveCheckoutSession(buyer, seller, orchestrationContext); // orchestrationContext has SetEC and createCheckoutSession results
  orchestrationContext.storeResponse(ApproveCheckoutSession, ApproveXOSessionResponse);
  
  // call api and store result
  GetECResponse = ECHandler.getEC(seller, orchestrationContext); //orchestrationContext has SetEC, createCheckoutSession and approveCheckoutSession results
  orchestrationContext.storeResponse(GetEC, GetECResponse);
  
  // call api and store result
  DoECResponse = ECHandler.doEC(DoECTemplate.SOME_DOEC_TEMPLATE, seller, orchestrationContext); //orchestrationContext has SetEC, createCheckoutSession, approveCheckoutSession and GetEC results
  orchestrationContext.storeResponse(DoEC, DoECResponse); // We could technically do without this line
}
```

While the test has become a little verbose, we're bringing in a lot more sanity to the handler method interfaces which is a plus. Using this approach it becomes clear that the createCheckoutSession() method is no longer doomed. It has a simple way of reaching into the orchestrationContext and finding the preceding SetEC response, from which it can extract the ec-Token and "apply" that as the cart_id in the request payload. The same goes for the approveCheckoutSession where we can extract the ec-Token to apply in the URI itself (but not the request body which is usually an empty json). If you think about getEC(), all it needs is the ec-Token to get a payerID from EC. DoEC relies on the payerID as well as the ec-Token, and since it has the apiResponse handy, this should be taken care of as well.

So once again, let's recap the exercise so far:
* We redesigned our code so that there are distinct endpoint based "handlers" (ex: ExpressCheckoutHandler, CheckoutSessionHandler)
* We created separate request mappers (ex: ExpressCheckoutRequestMapper) that react to payloadTemplates supplied by tests
* We captured each api's response in an orchestrationContext and provided it to each handler so that the handler's requestMapper can leverage it when needed.

Hold on. We've covered all this stuff without talking about the main purpose of writing the test which is performing the assertions. None of this matters unless we can accommodate the assertions/validations in a meaningful way.

For the purposes of simplicity, let's assume that we're just interested in validating 2 things: a successful approveCheckoutSession and a successul DoEC. In reality, what does success mean? Without getting too meta, success in the approveCheckoutSession response could literally translate to 2 things:
 * response statusCode is OK (200)
 * JSON representation of the response contains a field called "payment_approved" which is set to "true"

Success in the case of DoEC could mean the following 2 things:
 * String representation of the response has a field "ACK" that should be "Success"
 * String representation of the response has a field called "PAYMENTINFO_0_PAYMENTSTATUS" that should be "Completed"

You may recall that our handlers did 3 things: delegate requestMapping, call api, validateResponse. If they need to validateResponse, it's pretty clear that our tests need to pass the assertions to be performed into the handler interface. From the above "successful assertion checklist", it's clear that our definition of an assertion is providing a specific field and stating that it should be a specific value. If we're able to pass this data into the handler, then it seems natural that the handler would be able to collect the actual response, locate the "field" we're interested in and finally check whether its value is what we intended. Basically, we need to capture these "assertions" as a list of keyValue pairs. Perhaps our code for constructing these assertions would look a little like:

```java
public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  
  // validation construction
  Map<Field, ExpectedValue> approveXOSessionAssertionMap = new HashMap<Field, ExpectedValue>();
  approveXOSessionAssertionMap.put("status", "200");
  approveXOSessionAssertionMap.put("payment_approved", "true");
  
  Map<Field, ExpectedValue> doECAssertionMap = new HashMap<Field, ExpectedValue>();
  doECAssertionMap.put("ACK", "Success");
  doECAssertionMap.put("PAYMENTINFO_0_PAYMENTSTATUS", "Completed");
  
  ...
}
```
Capturing the expected values as strings makes sense, and putting these key value pairs in a map is understandable. But what about this notion of fields? The above example seems to indicate a String based field, which is fine but feels incomplete. That's because we need something at the other end (i.e. the handler layer) to actually "understand" how to locate these fields in the response. So, a fuller picture might look like:

```java
class MyTests {

  public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  
  // validation construction
  Map<Field, ExpectedValue> approveXOSessionAssertionMap = new HashMap<Field, ExpectedValue>();
  approveXOSessionAssertionMap.put("TheDamnApiResponseStatus", "200");
  approveXOSessionAssertionMap.put("payment_approved", "true");
  
  OrchestrationContext orchestrationContext = new OrchestrationContext();
  // assume SetEC and createCheckoutSession apis have been called
  ApproveXOSessionResponse = XOSessionHandler.approveCheckoutSession(buyer, seller, orchestrationContext, approveXOSessionAssertionMap);
  orchestrationContext.storeResponse(ApproveCheckoutSession, ApproveXOSessionResponse);
  ...
  }
}

class CheckoutSessionHandler {

  public approveCheckoutSession(PayPalUser buyer, PayPalUser seller, 
          OrchestrationContext orchestrationContext, Map<Field, ExpectedValue> assertionMap) {
      
    // map the request
    Json request = CheckoutSessionRequestMapper.mapApproveCheckoutSession();
    // call the api
    Response response = callApi(request);
    // validate the response
    validateResponse(response, assertionMap);
  }
  
  private validateResponse(Response response, Map<Field, ExpectedValue> assertionMap) {
      
    for (Field field : assertionMap.keySet()) {
      if (field == "payment_approved"){
         String actualFieldValue = response.parseThisField(field);
         String expectedFieldValue = assertionMap.get(field);
         assert(actualFieldValue, expectedFieldValue);
      } else if (field == "TheDamnApiResponseStatus") {
         // "TheDamnApiResponseStatus" is just verbose alias for response.getStatusCode()
         String actualFieldValue = response.getStatusCode();
         String expectedFieldValue = assertionMap.get(field);
         assert(actualFieldValue, expectedFieldValue);
      }
      ...
    }
  }
}
```
Now we're meaningfully honoring the assertionMap that we populated in the test and our supplied "field" has a corresponding map in the validateResponse() method. As the example illustrates, you can name the field whatever you like __as long as__ it has a correct mapping in the validateResponse method. As a side note, we could once again abstract out the validateResponse method into a separate class that strictly deals with responseValidation. Perhaps we could call it CheckoutSessionResponseValidator for the CheckoutSession endpoint? Given this, our test code can now look something like:

```java

public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  
  // validation construction
  Map<Field, ExpectedValue> approveXOSessionAssertionMap = new HashMap<Field, ExpectedValue>();
  approveXOSessionAssertionMap.put("status", "200");
  approveXOSessionAssertionMap.put("payment_approved", "true");
  
  Map<Field, ExpectedValue> doECAssertionMap = new HashMap<Field, ExpectedValue>();
  doECAssertionMap.put("ACK", "Success");
  doECAssertionMap.put("PAYMENTINFO_0_PAYMENTSTATUS", "Completed");
  
  // orchestration
  OrchestrationContext orchestrationContext = new OrchestrationContext();
  SetECResponse = ECHandler.setEC(SetECTemplate.SIMPLE_SALE_USD, seller, orchestrationContext, null); // the last parameter is null because we didn't create an assertionMap for setEC
  orchestrationContext.storeResponse(SetEC, SetECResult);
  
  CreateXOSessionResponse = XOSessionHandler.createCheckoutSession(buyer, seller, orchestrationContext, null);
  orchestrationContext.storeResponse(CreateCheckoutSession, CreateXOSessionResponse);
  
  ApproveXOSessionResponse = XOSessionHandler.approveCheckoutSession(buyer, seller, orchestrationContext, approveXOSessionAssertionMap);
  orchestrationContext.storeResponse(ApproveCheckoutSession, ApproveXOSessionResponse);
  
  GetECResponse = ECHandler.getEC(seller, orchestrationContext, null);
  orchestrationContext.storeResponse(GetEC, GetECResponse);
  
  DoECResponse = ECHandler.doEC(DoECTemplate.SOME_DOEC_TEMPLATE, seller, orchestrationContext, doECAssertionMap);
  orchestrationContext.storeResponse(DoEC, DoECResponse);
}
```

Fun stuff, but all this was mostly pseudocode. What would this use-case look like if a test was authored using FilterCoffee? Turns out, it's not that different:

```java
public void endToEndTestRealCode() {
  // orchestration assuming buyer/seller have been created
  
  List<ValidationUnit> approveXOSessionValidations = 
    validationUtility.build(PAYMENT_APPROVED, "true",
          STATUS_CODE, 200);
  
  List<ValidationUnit> doECValidations = 
    validationUtility.build(ACK, "Success",
          PAYMENTINFO_0_PAYMENTSTATUS, "Completed");
  
  filterCoffee.orchestrate(
      ExpressCheckout.Set, SetECTemplate.SIMPLE_SALE_USD,
      CheckoutSession.CREATE,
      CheckoutSession.APPROVE, approveXOSessionValidations,
      ExpressCheckout.GET,
      ExpressCheckout.DO, DoECTemplate.SIMPLE_SALE_WITH_INSURANCE_AND_SHIPPING_DISC, doECValidations,
      buyer, seller);
}
```

Let's analyze this for a moment. At first glance, some things stand out:
* There's an orchestrate method that's taking in a number of parameters
* Api's have been listed out using enums (ex: ExpressCheckout.SET, CheckoutSession.APPROVE etc)
* PayloadTemplates have been listed out using enums (ex: SetECTemplate.SIMPLE_SALE_USD)
* There's a validation utility that's taking in the fields and expected values
* Fields for validations are using enums, the validations are a list instead of a map
* Buyer and Seller have been passed once without any repetition

But how does this relate to the incremental design improvements we introduced through the above exercise? It's actually remarkably similar underneath: Instead of "handlers" we have "serviceImpls". Instead of "labels" we have "payloadTemplates". Instead of generic Apis we have "endpoints". Instead of assertionMaps we have "validationUnits". There is an in-built "orchestrationContext" that is constructed and passed around on each orchestration. You can expect a similar interaction where the test code is delegated to the appropriate serviceImpl, which in turn delegates to the appropriate requestMapper, calls the api via a REST/ASF client, and finally validates the response. The only new area will be the design that helps make the orchestrate(Object... args) interface possible, so the next section will focus on explaining FilterCoffee's basic building blocks.

###An overview of atomic units: ApiUnit, ValidationUnit, ExecutionUnit
Let's start with something simple. If you're trying to talk to an Api, what are the minimum possible things you need? For starters, you would need to know what api exactly you're trying to call. In the context of a RESTful api call, this means knowing the HTTP verb (ex: GET/POST etc) along with the endpointURL (ex: /v1/payments/checkout-sessions on a stage with the right port). In the context of an ASF call, this means knowing the exact api "method" (ex: planpaymentv2 via the MoneyPlanning client interface). Without getting too pedantic, we can just say that the first requirement for anyone to call an api is know what the "apiMethod/endpoint" is. Next, depending on the api requirements, there could be an associated request payload that you have to supply. Also, it's important to declare who *you* are as a client in charge of triggering the api (usually a merchant/seller). Since we're in the business of enabling e-commerce, it's not unusual to have a "buyer" purchasing something from the said merchant/seller.

So in the context of a transaction, all we really need to successfully call an api is: 
   * Api method (always)
   * Request payload (sometimes)
   * Seller (always)
   * Buyer (sometimes)

These fields together constitute an [ApiUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/source/ApiUnit.java) in FilterCoffee. It's just a simple POJO abstraction that captures all the necessary details required to "call" an api. [Note: You'll notice that the ApiUnit has some additional metadata such as clientChannel and headerInterface. This is to handle some symphony-specific business logic that influences request/header construction]

While this is all great news to "capture" necessary details neatly to "call" an api, what about being able to "test" the response? At its most basic level, "testing" can be thought of asserting whether an actual value is the same as the expected value. But actual value of what exactly? It usually helps to have the smallest level of granularity, and since any response can be visualized as a composition of its subfields, the simplest way to capture an "expectation" is to outline the "field" you're interested in, along with its "expectedValue". This is exactly what a [ValidationUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/source/ValidationUnit.java) is: a simple POJO capturing the expectedValue of your desired field in a response.

Finally, an [ExecutionUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/source/ExecutionUnit.java) is nothing more than a POJO that encapsulates an ApiUnit and a list of ValidationUnits.

If you think about our exercise where we incrementally built our test, the orchestration is nothing more than a list of Apis to be executed along with their respective validations. For this to work, we need to consider building an alias (as we did for PayloadTemplates) for the endpoint itself. So the old test could be rewritten as:

```java
class MyTests {

  public void endToEndTestPseudoCode() {
  // orchestration assuming buyer/seller have been created
  // assume validations have been constructed
  
  ExecutionUnit setECExecUnit = buildExecUnitForEC(ExpressCheckout.SET, SetECTemplate.SIMPLE_SALE_USD, buyer, seller, null);
  ExecutionUnit createXOSessionExecUnit = buildExecUnitForXOSession(CheckoutSession.CREATE, null, buyer, seller, null); // assume that buildExecUnitForXOSession has been created
  ExecutionUnit approveXOSessionExecUnit = buildExecUnitForApproveXOSession(CheckoutSession.APPROVE, null, buyer, seller, approveXOSessionAssertionMap);
  ExecutionUnit doECExecUnit = buildExecUnitForEC(ExpressCheckout.DO, DoECTemplate.SOME_DOEC_TEMPLATE, buyer, seller, doECAssertionMap);
  
  ...
  }
  
  public ExecutionUnit buildExecUnitForEC(ExpressCheckout ecApiMethod, SetECTemplate ecTemplate, PayPalUser buyer, PayPalUser seller, Map<Field, ExpectedValue> assertionMap) {
    ExecutionUnit execUnit = new ExecutionUnit();
    ApiUnit apiUnit = new ApiUnit();
    apiUnit.setApiMethod(ecApiMethod);
    apiUnit.setPayload(ecTemplate);
    apiUnit.setBuyer(buyer);
    apiUnit.setSeller(seller);
    // assume some utility that converts a map to list
    List<ValidationUnit> listOfValidations = getListFromMap(assertionMap);
    execUnit.setApiUnit(apiUnit);
    execUnit.setValidations(listOfValidations);
    return execUnit;
  }
}

// alias for EC-endpoints
enum ExpressCheckout {
  SET,
  GET,
  DO;
}

enum CheckoutSession {
  CREATE,
  APPROVE;
}
```
The code above is incomplete, but it communicates the idea that each api invocation along with its validation can be captured in an executionUnit. This process of individually building executionUnits is naturally very cumbersome; wouldn't it be nice if if there was a neat utility that did this grunt work on our behalf? This is a good opportunity to review the filterCoffee code and see what happens the moment you call orchestrate.

###A quick tour under the hood

This section assumes a familiarity with what an ExecutionUnit is, and flows better if you have read the section on "arriving at the design".

```java
class FilterCoffee {
  private static ExecutionUnitListBuilder executionUnitListBuilder = new ExecutionUnitListBuilder();
  
  public void orchestrate(Object... apiSequence) {
    // This utility takes in a variable number of arguments and packages them as a list of executionUnits
    final List<ExecutionUnit> execUnitList = executionUnitListBuilder.build(apiSequence);
    schedule(execUnitList);
  }

  
  private void schedule(final List<ExecutionUnit> executionList) throws Exception {
    OrchestrationContext orchestrationContext = new OrchestrationContext();
    // We loop over the list and execute each api one by one
    for (ExecutionUnit execUnit : executionList) {
        EndpointServiceFactory endpointServiceFactory = new EndpointServiceFactory();
        EndpointService endpointService = endpointServiceFactory.getEndpointService(execUnit);
        //Build any validations units that are needed to be built on the previous responses
        execUnit.buildDynamicValidations(orchestrationContext);
        // Once we figure out which endpoint service is responsible for handling the execution, we trigger the method
        Object result = endpointService.execute(execUnit, orchestrationContext);
        // Once we get the result, we add the response in
        orchestrationContext.storeResponse(execUnit.getApiUnit().getApiMethod(), result);
    }
  }
}
```
This code should make a lot more sense right now given how similar the pattern is compared to our refactored design in the old test. The [executionUnitListBuilder](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/util/ExecutionUnitListBuilder.java) takes care of the the grunt work of packaging the list of api parameters (method, payload etc) into a list of executionUnits. We have a local apiResponse map that is passed around in order to help some request mappers in doing their job appropriately. In filterCoffee, each endpoint has a corresponding endpointService that is responsible for handling api executions. If you look at the [CheckoutSession](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/CheckoutSession.java) enum, you'll find there's a method called [getEndpointService()](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/CheckoutSession.java#L36) which returns the appropriate serviceImpl (which in our earlier parlance, we called a "handler"). The next thing to look at in greater detail is the serviceImpl itself:

```java
    @Override
    public Object execute(final ExecutionUnit execUnit, final OrchestrationContext orchestrationContext) throws Exception {

        Endpoint apiMethod = execUnit.getApiUnit().getApiMethod();
        CheckoutSession csApi = (CheckoutSession) apiMethod;
        switch (csApi) {
        case APPROVE:
            return approveCheckoutSession(execUnit, orchestrationContext);
        case CHANGE_SHIPPING:
            return changeShippingAddress(execUnit, orchestrationContext);
        case CREATE:
            return createCheckoutSession(execUnit, orchestrationContext);
        case GET:
            return getCheckoutSession(execUnit, orchestrationContext);
        case UPDATE:
            return updateCheckoutSession(execUnit, orchestrationContext);
        default:
            throw new IllegalArgumentException("Unknown checkout-session endpoint called.");
        }
    }
    
    @Override
    public Object createCheckoutSession(final ExecutionUnit execUnit,
            final OrchestrationContext orchestrationContext) throws Exception {
        
        ObjectNode jsonRequest = csRequestMapper.mapCreateCheckoutSessionRequest(execUnit, orchestrationContext);
        String buyerAccNum = execUnit.getApiUnit().getBuyer().getAccountNumber();
        String merchantAccNum = execUnit.getApiUnit().getSeller().getAccountNumber();
        // Note that the ServiceKey may be influenced by the execUnit.getApiUnit().getClientChannel()
        // This would be the place to sort through the logic of how the clientChannel can influence it
        ServiceBridgeUnit serviceBridgeUnit = buildServiceBridgeUnit(jsonRequest, XOSessionURI, buyerAccNum,
                merchantAccNum, ServiceKey.PayPalSecurityHeader, null, null);
        Pair<JsonNode, StatusType> responsePair = post(serviceBridgeUnit);
        JsonNode jsonResponse = responsePair.getLeft();
        StatusType statusType = responsePair.getRight();
        csResponseValidator.validateCreateCheckoutSession(statusType.getStatusCode(), execUnit, jsonResponse);
        return jsonResponse;
    }
```
Here we simply dispatch to the appropriate method based on the apiMethod that is present in the executionUnit. Once dispatched, key parameters are extracted. For starters, there is a [CheckoutSessionRequestMapper](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/mapper/CheckoutSessionServiceRequestMapper.java) that comes into play in order to build the request. Next, we parse out parameters required to send out the request under a simple POJO called the [ServiceBridgeUnit](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/source/ServiceBridgeUnit.java). The key piece that handles the actual api call is in the post() method, which is inherited from the [ServiceBridge](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/bridge/ServiceBridge.java) base class. Once a response is received, it is dispatched to a [CheckoutSessionServiceResponseValidator](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/validator/CheckoutSessionServiceResponseValidator.java) which loops over the list of assertions that are embedded in the execUnit and compares the actual value to the expected value. Let's take a look at what happens in the ServiceBridge.

```java
  public class ServiceBridge {
  
    public Pair<JsonNode, StatusType> post(final ServiceBridgeUnit serviceBridgeUnit) throws Exception {
        return callApi(HttpMethod.POST, serviceBridgeUnit);
    }  
        
    private Pair<JsonNode, StatusType> callApi(HttpMethod httpMethod, final ServiceBridgeUnit serviceBridgeUnit)
            throws Exception {

        JsonNode request = serviceBridgeUnit.getJsonRequest();
        final SimpleJsonRestClient restClient = JsonClientFactory.getClient(serviceBridgeUnit);
        ServiceBridgeUtil.logRequest(httpMethod.toString(), request, restClient.getEndPointURL(), restClient
                .getHeaders().toString());
        JsonNode response = null;

        switch (httpMethod) {
        case POST:
            response = restClient.post(request);
            break;
        case GET:
            response = restClient.get();
            break;
        case PATCH:
            response = restClient.patch(request);
            break;
        default:
            break;
        }

        ServiceBridgeUtil.logResponse(httpMethod.toString(), response, restClient.getEndPointURL(), restClient
                .getStatus().getStatusCode(), restClient.getStatus().getReasonPhrase());
        ServiceBridgeUtil.logCalLink(restClient.getResponseHeaders());
        Pair<JsonNode, StatusType> responseAndStatusPair = Pair.of(response, restClient.getStatus());
        return responseAndStatusPair;
    }
  
  }
```
The most important part of this method is the invocation of the [restClient](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/bridge/client/SimpleJsonRestClient.java) where we pass the ServiceBridgeUnit POJO that contains all the information regarding seller, serviceKey etc. The rest is simple logging. The [JsonClientFactory](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/bridge/client/JsonClientFactory.java) is responsible for building a rest client with the appropriate headers, url, and port. It pulls most of this information off of a static json file called [filtercoffeeconfig.json](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/resources/filtercoffeeconfig.json) and mutates the URL based on the servicePath dictated by the serviceImpl. It mutates the headers based on the serviceKey selectively and embeds an accountNumber/Id wherever necessary.

Hopefully, this has given you an end to end understanding of the journey of a test through some of the FilterCoffee rails. Below is an elaborate list of TODO's that I didn't get time to get to, so hopefully we'll cover some of this in the brown bag. If there's any feedback you have about this tutorial or filterCoffee in general, feel free to shoot an email to sahjain@paypal.com

##Architectural Diagram
![](arch.png)

### Writing an end to end test

Let's start by opening the functional-tests project in you IDE, and navigate to the src/test/java folder. Here you will find a number of packages, and since we're interesting in locating an appropriate location to park our tests, we'd like to write a package prefixed by "filtercoffee.test". You will notice that there's a number of such packages like `com.paypal.test.filtercoffee.test.aries.guest` or `com.paypal.test.filtercoffee.test.prox.msp`. Similarly, let's create a package called `com.paypal.test.filtercoffee.test.illumination` since we're hoping that this exercise will illuminate us by the time we're done. Let's write a class called `FilterCoffeeTests`. You should have something like the following:

```java
package com.paypal.test.filtercoffee.test.illumination;

public class FilterCoffeeTests {

}

```
Great, so now we have a home to park our tests. If you haven't read the [original docs](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/README.md), please take 10mins and do so. This exercise will yield much more benefit if you've gone through that documentation.

Let's start with a basic prox use-case. As you may be aware, a simple prox flow entails the invocation of 3 APIs: POST/Cart, GET/funding-options, and finally a POST/Payment. Let's see how we can implement this in filterCoffee. First, we'll instantiate a filterCoffee object which will give us access to its `orchestrate` method. Next, we'll start building users that are relevant to this transaction. Let's call this test method `simpleProxEndToEndTest()`.

```java
package com.paypal.test.filtercoffee.test.illumination;

import org.testng.annotations.Test;

import com.paypal.test.filtercoffee.engine.FilterCoffee;
import com.paypal.test.filtercoffee.util.UserBaseUtil;
import com.paypal.test.jaws.ppo.Country;
import com.paypal.test.jaws.ppo.CreditCardType;
import com.paypal.test.jaws.ppo.PPAccountType;
import com.paypal.test.jaws.ppo.PayPalUser;
import com.paypal.types.Currency.Info;

public class FilterCoffeeTests {
    
    private FilterCoffee filterCoffee = new FilterCoffee();
    private static final String masterProxFacilitatorEmail = "spoigai-de1-seller@paypal.com";
    private static final String masterDESellerEmail = "sbhadra-de-biz@paypal.com";
    
    @Test
    public void simpleProxEndToEndTest() throws Exception {
        
        PayPalUser buyer = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.BUSINESS, 
                Country.DE, Info.Euro, null, CreditCardType.VISA);
        PayPalUser facilitator = UserBaseUtil.getLegacyUser(masterProxFacilitatorEmail);
        PayPalUser seller = UserBaseUtil.getLegacyUser(masterDESellerEmail);
        
    }
    
}
```
Let's pause here and see what's happening in the code so far. It's clear that we haven't used filterCoffee's orchestration yet, but that is intentional to first focus on the user construction. For `buyer` you'll notice that we're going through a method exposed by a class called `UserBaseUtil` that takes in all the attributes to create a DE buyer of type BUSINESS with a VISA card that holds Euro currency. If you step into the code, you'll notice that the UserBaseUtil is simply a class that is a wrapper around another class called `UserUtility` to provide convenient abstractions for user creation. The reason we land in `UserUtility` ultimately is that it hosts intelligent caching to avoid spending too many cycles re-creating a user with the same attributes. This caching yields huge benefits, and in the near future UserUtility will not be a local library call, but instead talk to the `UMS` service that was conceived during an earlier Hackathon. The good news is that the UserBaseUtil will never need to change regardless of the mechanism for creating/retrieving users.

Next, you'll find that for the `facilitator` and `seller`, we skip the actual manual construction of the account, but rather go via a `getLegacyUser()` method. The reason this is the case is that these accounts are "master" accounts available on every stage. More importantly, these accounts have a number of different settings and flags that are unavailable to tap into (reliably) through any API calls. At some point in the future, this "special" account construction will also be available through a series of DB queries and API calls in the background, but for now this will suffice.

If the user construction is clear so far, let's move on to the actual orchestration of APIs. As you may have read in the original filterCoffee docs, we use aliases heavily. This means that there is an alias for calling POST/Cart, and there's also an alias for supplying the appropriate payload to accompany the HTTP request. For the GET/funding-options API, there is no payload, so we just end up using the alias for calling the API itself. Finally, for the POST/Payment API, we need to supply a payload alias of some sort that informs the relevant request mapper how it should "build" the appropriate payload. As you are aware, different use-cases mandate different payloads. For example, an Aries use-case (via the DoEC migration) would NOT supply a fundingOptionId (since symphony stores the selected funding option id as a byproduct of the approveCheckoutSession api). However, in a standard Prox use-case, the payload needs to have the "selected fundingOptionId" since there is no explicit approval step. Here's what an orchestration could look like:

```java
package com.paypal.test.filtercoffee.test.illumination;

import org.testng.annotations.Test;

import com.paypal.test.filtercoffee.endpoint.resource.Cart;
import com.paypal.test.filtercoffee.endpoint.resource.Payment;
import com.paypal.test.filtercoffee.engine.FilterCoffee;
import com.paypal.test.filtercoffee.source.ClientChannel;
import com.paypal.test.filtercoffee.template.payload.CartPayloadTemplates.CreateCartTemplate;
import com.paypal.test.filtercoffee.template.payload.PaymentPayloadTemplates.CreatePaymentTemplate;
import com.paypal.test.filtercoffee.util.UserBaseUtil;
import com.paypal.test.jaws.ppo.Country;
import com.paypal.test.jaws.ppo.CreditCardType;
import com.paypal.test.jaws.ppo.PPAccountType;
import com.paypal.test.jaws.ppo.PayPalUser;
import com.paypal.types.Currency.Info;

public class FilterCoffeeTests {
    
    private FilterCoffee filterCoffee = new FilterCoffee();
    private static final String masterProxFacilitatorEmail = "spoigai-de1-seller@paypal.com";
    private static final String masterDESellerEmail = "sbhadra-de-biz@paypal.com";
    
    @Test
    public void simpleProxEndToEndTest() throws Exception {
        
        PayPalUser buyer = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.BUSINESS, 
                Country.DE, Info.Euro, null, CreditCardType.VISA);
        PayPalUser facilitator = UserBaseUtil.getLegacyUser(masterProxFacilitatorEmail);
        PayPalUser seller = UserBaseUtil.getLegacyUser(masterDESellerEmail);
        
        filterCoffee.orchestrate(
                Cart.CREATE, CreateCartTemplate.SIMPLE_USD,
                Cart.GET_FO,
                Payment.CREATE, CreatePaymentTemplate.REQUEST_WITH_FO_ID,
                ClientChannel.ProxMemberRememberMe, buyer, facilitator, seller);
        
    }
    
}
```
Let's pause and look at what's going on here. The first parameter in the orchestrate method is a `Cart.CREATE`. This is the alias we were referring to earlier. If you click on the `Cart` enum you'll notice that it has a number of such api calls. In fact, all the fields in Cart are modeled based on the [CartResource](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/rest-model/src/main/java/com/paypal/platform/payments/api/rest/resource/CartResource.java) in the original symphony code. This is precisely the reason the Cart enum lives in the `com.paypal.test.filtercoffee.endpoint.resource` package, because it is an endpoint resource that serves as an alias to call the api.

Since calling an endpoint like Cart.CREATE is meaningless without an actual payload that you want to supply as a client, we'd like to survey what are the available cart payload templates. For this, feel free to hit Cmd+Shift+T and search for "CartPayloadTemplates". You'll stumble upon the [CartPayloadTemplates](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/payload/CartPayloadTemplates.java) class that lists all payload templates relevant to the cart-based endpoints. You'll see that there is a [CreateCartTemplate](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/payload/CartPayloadTemplates.java#L22), and a PatchCartTemplate, both of which implement a simple PayloadTemplate interface. CreateCartTemplate looks like a good fit, so let's take a moment to survey all the different flavors of payloads that we could end up using. The first one, [SIMPLE_USD](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/payload/CartPayloadTemplates.java#L24) looks promising since it points to a [single purchase unit containing a couple of items with an intent of sale](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/resources/prox/prox-create-cart-request-USD.json). Looks like we've gathered the relevant information to call the Cart endpoint with relative ease.

Moving onto the next api, we'd like to call the GET/funding-options endpoint. Specifically, since we're trying to get the funding-options based on the cart, we can find the relevant endpoint in the Cart enum called [Cart.GET_FO](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/Cart.java#L23). Since this is a GET call, there isn't any associated payload so we don't have hunt for anything suitable in the CartPayloadTemplates class.

Finally, using a similar pattern as earlier, we attempt to do a POST/Payment by looking at the [Payment](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/Payment.java) enum. Since it is modeled after the PaymentResource, we can expect to find something that does a createPayment which in this case is a [Payment.CREATE](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/Payment.java#L17). Now as discussed earlier, to call the POST/Payment endpoint, we need an associated payload. In this, we would like something that has a fundingOptionId since it is a Prox use-case. Once again, we hit Cmd+Shift+T and search for "PaymentPayloadTemplates". You arrive at a [class with the same name](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/payload/PaymentPayloadTemplates.java) (see the pattern here?) which lists out the various flavors of payloadTemplates that are relevant to the payment endpoint. We'll navigate to the [CreatePaymentTemplate](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/payload/PaymentPayloadTemplates.java#L25) and pick the REQUEST_WITH_FO_ID as our desired alias.

Great, so now we've completely sorted out our api endpoints which is great. But we'd also like to introduce the parties relevant to this transaction that we had created earlier (buyer, facilitator, seller). But specifically, we'd also like to communicate that this is a prox use-case. Prox has a number of different flavors ranging from rememberMe, nonRememberMe and guest. We'll navigate to the [ClientChannel](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/source/ClientChannel.java#L28) and pick up the ProxRememberMe enum.

Now that you understand how appropriately supply the information required for making the different API calls, let's take a look at the code once again.

```java
    @Test
    public void simpleProxEndToEndTest() throws Exception {
        
        PayPalUser buyer = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.BUSINESS, Country.DE,
                Info.Euro, null, CreditCardType.VISA);
        PayPalUser facilitator = UserBaseUtil.getLegacyUser(masterProxFacilitatorEmail);
        PayPalUser seller = UserBaseUtil.getLegacyUser(masterDESellerEmail);
        
        filterCoffee.orchestrate(
                Cart.CREATE, CreateCartTemplate.SIMPLE_USD,
                Cart.GET_FO,
                Payment.CREATE, CreatePaymentTemplate.REQUEST_WITH_FO_ID,
                ClientChannel.ProxMemberRememberMe, buyer, facilitator, seller);
        
    }
```

If you read the [original docs](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/README.md#arriving-at-the-design), then you know what's happening inside the orchestrate method. Essentially, a list of ExecutionUnits are being built and looped with a method-local transaction context called the orchestrationContext. You'll notice that as an enum implementing an Endpoint, the [Cart](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/Cart.java#L40) actually has an associated [CartServiceImpl](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/impl/CartServiceImpl.java) which is responsible for [making the api call](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/impl/CartServiceImpl.java#L92). It does this by building a request (using the payload you provided) and underneath orchestrates all the oauth calls that are necessary for assembling the appropriate headers.

This is a good moment to run the test. We recommend that you set global VM arguments such that running tests is as simple as right clicking on the method name, clicking on Run As -> TestNG test. In order to accomplish this, click on your IDE's preferences (In STS installations on a Mac OSX, this means you click on Spring Tool Suite -> Preferences). Here you will find a number of fields, scroll to the TestNG option on the left and expand the drop down. You should now see a Run/Debug option, so click on it and enter the following JVM args: __-DJAWS_HOSTNAME=stage2XXXX.qa.paypal.com__. Note that you're welcome to use whatever stage you please as long as you have passwordless setup for that stage. If you're confused about what passwordless access is, we recommend you read the setup instructions on the original document.

You should notice that for about a minute, very sparse console logging happens. This is due to the fact that the initial test data requires user creation which takes time (that's why we introduced caching underneath!) Soon you should notice some oauth api calls, followed by the intended api calls like POST/Cart. If you scroll towards the end of the console, you'll notice something like the following:

```java
Passed 1 of 3:  *****POST CART - STATUS_CODE Verified*****  
Passed 2 of 3:  *****GFO - STATUS_CODE Verified*****  
Passed 3 of 3:  *****POST PAYMENT - STATUS_CODE Verified*****
```
This brings us to an important part of the test, which is the actual assertions that we'd like applied. You may have made an astute observation several minutes ago that we had just authored a test that, well, didn't really "test" anything. It merely orchestrated 3 apis but that was really it. The console dump above demonstrates that even if you don't provide any assertions, basic ones are made on your behalf like checking the typical status code of the relevant apis (201 for POST/Cart, 200 for GET/funding-options).

So let's proceed by constructing and supplying better assertions! After reading the original docs, you should be familiar with what a ValidationUnit is (essentially a POJO that holds a field along with its expected value). So the quality of our tests are directly correlated with the quality of the validationUnits that we supply into the test. Let's take a look at how to construct the validationUnits for POST/Cart. The POST/Cart api is such that much of the payload is actually echoed back in the response as a Cart resource. So one very obvious way to start writing simple assertions is manually inspecting the payload.

Let's resume this journey after a brief segue into how to identify the specific fields of interest. If you want to build assertions, then you ought to be able to confidently say things like "I want my amount total to be $5". Here, "5" could be an expected value, but what about the amountTotal field? Once again, we fall back on using aliases to express the field of interest. Since this is the Cart resource, then we would like to see what kind of cart resource fields are available. Feel free to hit Cmd+Shift+T and search for "CartResourceFields" and you should stumble upon [this class](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/field/alias/CartResourceFields.java). You'll notice a number of fields, including one for `AMT_TOTAL`. Notice that it has a corresponding jsonPath `purchase_units[x]/amount/total`. This is totally made up for the pure expressive purposes of conveying where this field lies. We could have set the jsonPath to something like `you know, where the amount total usually is bro.` but that wouldn't be terribly helpful now would it. Please note that this enum is simply a fieldAlias; it provides simple ways to identify a number of fields.

Cool, so we now know that there's a way for us to capture specific fields and set their expected value. But there's another issue. You may have made an observation earlier that "Hey! I see that AMT_TOTAL points out that it is for something in purchase_units[x]. How can I convey correctly what exact purchaseUnit index this AMT_TOTAL belongs in?". This is where alias fall short. An enum is great for many things, but now there's clearly a need for it to capture more metadata about the exact location of the purchaseUnit which it currently doesn't have and unhelpfully labels as "x". If only there were a more serious cart field that encapsulated all this rich metadata along with the relevant fieldAlias. Once again, hit Cmd+Shift+T and search for "CartField" and you'll stumble upon [this class](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/field/CartField.java). You'll notice that it holds a reference to a CartResourceField as well as all the metadata for purchaseUnitIndex, itemIndex etc. Going back to the CartResourceFields enum you will notice a [reference to this CartField](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/field/alias/CartResourceFields.java#L58). By virtue of being a FieldAlias, there needs to be a corresponding reference to something implementing a Field. Since our validationUnit POJO accepts only an object of type "Field" and an expected value of type "String", we can write our validation as follows:

```java
List<ValidationUnit> cartValidations = new ArrayList<ValidationUnit>();
cartValidations.add(new ValidationUnit(CartResourceFields.AMT_TOTAL.getField(), "41.15"));
```
The expectedValue in this validation is 41.15 (we got this number by inspecting our [original payload](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/resources/prox/prox-create-cart-request-USD.json)) and the specific field in question is AMT_TOTAL. The puIndex was intialized to 0 in the background (feel free to look at the implementation of the CartResourceFields enum to verify this). This is a reasonable start, but frankly we'd like to validate so much more. What else? Well, let's scan our payload and see what other kinds of fields we can apply. AMT_SHIPPING, AMT_SUBTOTAL, AMT_TAX, INTENT etc. The list is pretty long and we could (if we wanted) end up doing something like this:

```
List<ValidationUnit> cartValidations = new ArrayList<ValidationUnit>();
cartValidations.add(new ValidationUnit(CartResourceFields.AMT_TOTAL.getField(), "41.15"));
cartValidations.add(new ValidationUnit(CartResourceFields.AMT_SHIPPING.getField(), "11.00"));
cartValidations.add(new ValidationUnit(CartResourceFields.AMT_SUBTOTAL.getField(), "30.00"));
cartValidations.add(new ValidationUnit(CartResourceFields.AMT_TAX.getField(), "0.15"));
cartValidations.add(new ValidationUnit(CartResourceFields.INTENT.getField(), "sale"));
```
This is pretty laborious, especially considering how tightly related this is to the actual payload itself. Moreover, if we wanted to be clever in the future and have our test with data provider where we hammer away mutliple CreateCartTemplates, then this manual construction would break down. Can we do better? Seems like there's an aching need for some kind of utility that could simply inhale the createCartTemplate and magically autogenerate a list of validations for us. A cool bonus would be if it handled all the funky metadata management as well so that we can have meaningful and accurate validations.

Here we introduce the notion of a [validationUtility](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/util/ValidationUtil.java). To use it, all you have to do is pass the particular endpoint for which you're interested in creating validations (in our case, this is Cart.CREATE). Then, just pass in the template that is relevant (in our case, this is the CreateCartTemplate.SIMPLE_USD). This would end up looking something like this:

```java
    @Test
    public void simpleProxEndToEndTest() throws Exception {

        PayPalUser buyer = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.BUSINESS, Country.DE,
                Info.Euro, null, CreditCardType.VISA);
        PayPalUser facilitator = UserBaseUtil.getLegacyUser(masterProxFacilitatorEmail);
        PayPalUser seller = UserBaseUtil.getLegacyUser(masterDESellerEmail);

        List<ValidationUnit> cartValidations = validationUtility.build(Cart.CREATE, CreateCartTemplate.SIMPLE_USD);

        filterCoffee.orchestrate(
                Cart.CREATE, CreateCartTemplate.SIMPLE_USD, cartValidations,
                Cart.GET_FO,
                Payment.CREATE, CreatePaymentTemplate.REQUEST_WITH_FO_ID,
                ClientChannel.ProxMemberRememberMe, buyer, facilitator, seller);

    }
```
So we've got a relatively sleek way to build these validations and you'll notice that we've tucked it in the first line of the orchestration method. Does this even work? Well, we recommend that you use this code and run the test! You should end up seeing this towards the end of your console dump:

```java
Passed 1 of 28:  *****POST CART - STATUS_CODE Verified*****  
Passed 2 of 28:  *****POST CART - INTENT Verified*****  
Passed 3 of 28:  *****POST CART - AMT_TOTAL Verified*****  
Passed 4 of 28:  *****POST CART - AMT_CURRENCY Verified*****  
Passed 5 of 28:  *****POST CART - AMT_SUBTOTAL Verified*****  
Passed 6 of 28:  *****POST CART - AMT_SHIPPING Verified*****  
Passed 7 of 28:  *****POST CART - AMT_TAX Verified*****  
Passed 8 of 28:  *****POST CART - ITEM_CURRENCY Verified*****  
Passed 9 of 28:  *****POST CART - ITEM_NAME Verified*****  
Passed 10 of 28:  *****POST CART - ITEM_PRICE Verified*****  
Passed 11 of 28:  *****POST CART - ITEM_QTY Verified*****  
Passed 12 of 28:  *****POST CART - ITEM_SKU Verified*****  
Passed 13 of 28:  *****POST CART - ITEM_CURRENCY Verified*****  
Passed 14 of 28:  *****POST CART - ITEM_NAME Verified*****  
Passed 15 of 28:  *****POST CART - ITEM_PRICE Verified*****  
Passed 16 of 28:  *****POST CART - ITEM_QTY Verified*****  
Passed 17 of 28:  *****POST CART - ITEM_SKU Verified*****  
Passed 18 of 28:  *****POST CART - ADDR_CITY Verified*****  
Passed 19 of 28:  *****POST CART - ADDR_COUNTRY_CODE Verified*****  
Passed 20 of 28:  *****POST CART - ADDR_LINE1 Verified*****  
Passed 21 of 28:  *****POST CART - ADDR_PHONE Verified*****  
Passed 22 of 28:  *****POST CART - ADDR_POSTAL_CODE Verified*****  
Passed 23 of 28:  *****POST CART - ADDR_RECIPIENT_NAME Verified*****  
Passed 24 of 28:  *****POST CART - CANCEL_URL Verified*****  
Passed 25 of 28:  *****POST CART - RETURN_URL Verified*****  
Passed 26 of 28:  *****GFO - STATUS_CODE Verified*****  
Passed 27 of 28:  *****POST PAYMENT - STATUS_CODE Verified*****  
Passed 28 of 28:  *****POST PAYMENT - AMT_VALUE Verified*****
```
How exciting! We've tapped into a utility that parsed the json file referenced by the CreateCartTemplate.SIMPLE_USD and generated a list of 25 additional validations. If we had a "larger" payload with more details like postback data, then these too would have been parsed by the validationUtility. Let's move onto building validations for the next api, GET/funding-options.

Now recall the way you built the buyer account. You added a single fundingInstrument (CreditCard VISA) so needless to say, the default funding option you expect should have that specific instrument. Since the buyer doesn't have any other fundingInstruments then it makes sense that there won't be any "alternate" fundingOptions. Once again, a relatively simple and manual attempt at building the validations for something like this could be the following:

```java
List<ValidationUnit> gfoValidations = new ArrayList<ValidationUnit>();
gfoValidations.add(new ValidationUnit(FundingOptionsResourceFields.DEF_FUNDING_INSTRUMENT_TYPE.getField(), "PAYMENT_CARD"));
```
Once again, this isn't terrible, but it isn't hard to imagine a scenario where if we inject the funding instrument through a dataProvider. In that case, this validation no longer becomes consistent when we pass, say, a Bank account. There's wiggle room to do better. If you read the original docs, you would have stumbled upon this notion of a ValidationTemplate. The validationTemplate is something that succinctly captures a "situation" that has been prepared as per the test data. In this case, the buyer having a creditCard as the sole funding instrument is certainly something that would lead to a set of generic validations like the defaultFundingOption's fundingMode being INSTANT_TRANSFER and the fundingInstrument being PAYMENT_CARD along with the fact that alternateFundingOptions should be NULL.

Now let's pause and think before we rush to plug in some relevant validationTemplate. Why didn't we pursue the route of passing the VISA card as a "payload" and have the validationUtility build something on our behalf? After all, the VISA being an input to the test data could be thought of as a "payload", and a list ought to be generated on its behalf. While this would work just fine, it fails to capture a multitude of other scenarios. For example: what if you set up a test such that the seller "rejects/disables" payments from a VISA card. That would mean the getFundingOptions api would return some contingency, but our validations would naively expect a 201 with a PAYMENT_CARD in the response. In such a situation, it's easy to see how a validationTemplate could "capture" this rigged scenario and provide an alias to validate situation. It is for this reason we prefer going down the route of weilding a validationTemplate instead of just passing in the VISA card.

Now that we've warmed up to the idea of using some validationTemplate, the question is, where can we find one? Just as before, we must remember we're dealing with a Cart endpoint. In order to find the relevant validation templates for the cart endpoint, press Cmd+Shift+T and look for "CartValidationTemplates". You should be able to locate [this class](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/validation/CartValidationTemplates.java). Here you will find an enum of interest called [GetFundingOptionsValidationTemplate](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/validation/CartValidationTemplates.java#L5). Here you'll find a number of different scenarios. The one relevant to us is the `DEF_CC_WITHOUT_ALT` which is a shorthand expression for "Default fundingOption is a CreditCard with NO alternate options". Once you plug this into the test it ought to look a little like:

```java
    @Test
    public void simpleProxEndToEndTest() throws Exception {
        
        PayPalUser buyer = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.BUSINESS, Country.DE,
                Info.Euro, null, CreditCardType.VISA);
        PayPalUser facilitator = UserBaseUtil.getLegacyUser(masterProxFacilitatorEmail);
        PayPalUser seller = UserBaseUtil.getLegacyUser(masterDESellerEmail);

        List<ValidationUnit> cartValidations = validationUtility.build(Cart.CREATE, CreateCartTemplate.SIMPLE_USD);
        List<ValidationUnit> gfoValidations = validationUtility.build(Cart.GET_FO, GetFundingOptionsValidationTemplate.DEF_CC_WITHOUT_ALT);

        filterCoffee.orchestrate(
                Cart.CREATE, CreateCartTemplate.SIMPLE_USD, cartValidations,
                Cart.GET_FO, gfoValidations,
                Payment.CREATE, CreatePaymentTemplate.REQUEST_WITH_FO_ID,
                ClientChannel.ProxMemberRememberMe, buyer, facilitator, seller);

    }
```
Once you run your test, you'll find more validations that happen for the Cart.GET_FO endpoint. Here's a snippet of the assertions you should find in your console:

```java
Passed 26 of 30:  *****GFO - DEF_FUNDING_MODE Verified*****  
Passed 27 of 30:  *****GFO - DEF_FUNDING_INSTRUMENT_TYPE Verified*****  
Passed 28 of 30:  *****GFO - ALT_NULL Verified*****  
```
This is definitely an improvement, but can we do better? For starters, we haven't included a critical piece of information - the amount and currency! Now, you could manually append the list with the relevant information, but if we've learned anything from the createCartValidations is that life is short, and we're better off throwing the whole payload at the validationUtility so that it can parse it for the relevant information (which in this case is the specific amount and currency of the purchaseUnit). We can improve the validation construction as follows:

```java
List<ValidationUnit> gfoValidations = validationUtility.build(Cart.GET_FO, GetFundingOptionsValidationTemplate.DEF_CC_WITHOUT_ALT, CreateCartTemplate.SIMPLE_USD);
```
If you re-run your test, you'll notice that the AMT and CURRENCY are also validated once Cart.GET_FO is invoked. This definitely feels better once again. Can it be improved further? It sure can. After all, we haven't quite captured the exact creditCard type which was VISA. As of this writing, this is not implemented in filterCoffee. We will tackle this improvement later in this tutorial as an exercise.

Moving forward, we now arrive at the most important api which is POST/Payment. Being able to actually move money in a checkout is what pays us our salaries, so let's see if we can appropriately build the validations for this endpoint. Before we dive in, let's consider the kinds of things we'd like validated for POST/Payment. We'd like it to be "successful" (so, a 201). We'd also like it to echo relevant information like the AMT and CURRENCY. We'd also like to capture the fact that this was made using a CC, so that should be an INSTANT_TRANSFER along with the intent which was SALE. Following the same pattern as earlier, we can capture the AMT and CURRENCY by supplying the CreateCartTemplate. We can also use a [PaymentValidationTemplate](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/validation/PaymentValidationTemplates.java#L5) that captures the fact that it is an INSTANT_SALE. This would mean our validation for payment in the test might look a little like:

```java
List<ValidationUnit> payValidations = validationUtility.build(Payment.CREATE, CreateCartTemplate.SIMPLE_USD, PaymentValidationTemplate.INSTANT_SALE);
```
Use this in your test and run it. You should end up having close to 20 assertions applied for POST/Payment. Not bad for a 1 liner.

Let's get a little creative and throw in some extra api calls for good measure in our orchestration. Perhaps a "fuller" orchestration might look something like:

```java
    @Test
    public void simpleProxEndToEndTest() throws Exception {
        
        PayPalUser buyer = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.BUSINESS, Country.DE,
                Info.Euro, null, CreditCardType.VISA);
        PayPalUser facilitator = UserBaseUtil.getLegacyUser(masterProxFacilitatorEmail);
        PayPalUser seller = UserBaseUtil.getLegacyUser(masterDESellerEmail);

        List<ValidationUnit> cartValidations = validationUtility.build(Cart.CREATE, CreateCartTemplate.SIMPLE_USD);
        List<ValidationUnit> getCartValidations = cartValidations;
        List<ValidationUnit> gfoValidations = validationUtility.build(Cart.GET_FO,
                GetFundingOptionsValidationTemplate.DEF_CC_WITHOUT_ALT, CreateCartTemplate.SIMPLE_USD);
        List<ValidationUnit> payValidations = validationUtility.build(Payment.CREATE, CreateCartTemplate.SIMPLE_USD, PaymentValidationTemplate.INSTANT_SALE);
        List<ValidationUnit> getPayValidations = validationUtility.build(Payment.GET, CreateCartTemplate.SIMPLE_USD, GetPayValidationTemplate.PAYEE_EMAIL);

        filterCoffee.orchestrate(
                Cart.CREATE, CreateCartTemplate.SIMPLE_USD, cartValidations,
                Cart.GET, getCartValidations,
                // We get the eligible payments which are validated internally
                Cart.GET_EPM,
                Cart.GET_FO, gfoValidations,
                Payment.CREATE, CreatePaymentTemplate.REQUEST_WITH_FO_ID, payValidations,
                // Get the Sale subresource
                Sale.GET,
                // Get the Payment resource and ensure that the payeeEmail is the seller's
                Payment.GET, getPayValidations,
                // We want to make sure nothing changed in the cart after the payment
                Cart.GET, getCartValidations,
                ClientChannel.ProxMemberRememberMe, buyer, facilitator, seller);

    }
```
Now let's say that you'd like to PATCH the cart midway through the transaction. Once again, there's a straight forward pattern to follow. The relevant endpoint can be found in the Cart enum called Cart.UPDATE. Naturally, we'll need to provide an appropriate payloadTemplate for such a thing that can be found in the CartPayloadTemplates class. Let's pick the PatchCartTemplate.SIMPLE. Great so we're good to go right?

Not so fast! If you're going to patch the cart PRIOR to getting the fundingOptions, that can potentially change the AMT and any other number of things. Now it becomes important to be able to capture this patch while constructing validations for Cart.GET_FO, Payment.CREATE, as well as the final Cart.GET. Moreover, how in the world are we supposed to automatically build the validations for the PATCH/Cart api call itself? Fear not, the validationUtility comes to our rescue once again.

```java
    @Test
    public void simpleProxEndToEndTest() throws Exception {

        PayPalUser buyer = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.BUSINESS, Country.DE,
                Info.Euro, null, CreditCardType.VISA);
        PayPalUser facilitator = UserBaseUtil.getLegacyUser(masterProxFacilitatorEmail);
        PayPalUser seller = UserBaseUtil.getLegacyUser(masterDESellerEmail);

        List<ValidationUnit> cartValidations = validationUtility.build(Cart.CREATE, CreateCartTemplate.SIMPLE_USD);
        List<ValidationUnit> firstGetCartValidations = cartValidations;
        // We build the patchCart validations by providing the template and the original cart
        // This way the validationUtility can "apply" the patch onto the original cart payload and parse the resulting json
        List<ValidationUnit> patchCartValidations = validationUtility.build(Cart.UPDATE, CreateCartTemplate.SIMPLE_USD, PatchCartTemplate.SIMPLE);
        // Our gfoValidations are updated to include the patchCartTemplate
        List<ValidationUnit> gfoValidations = validationUtility.build(Cart.GET_FO,
                GetFundingOptionsValidationTemplate.DEF_CC_WITHOUT_ALT, CreateCartTemplate.SIMPLE_USD,
                PatchCartTemplate.SIMPLE);
        // Our payValidations are updated to include the patchCartTemplate
        List<ValidationUnit> payValidations = validationUtility.build(Payment.CREATE, CreateCartTemplate.SIMPLE_USD,
                PatchCartTemplate.SIMPLE, PaymentValidationTemplate.INSTANT_SALE);
        // Our getPayValidations are updated to include the patchCartTemplate
        List<ValidationUnit> getPayValidations = validationUtility.build(Payment.GET, CreateCartTemplate.SIMPLE_USD,
                PatchCartTemplate.SIMPLE, GetPayValidationTemplate.PAYEE_EMAIL);
        // Our secondGetCartValidations should be same as the cart when it was first patched
        List<ValidationUnit> secondGetCartValidations = patchCartValidations;

        filterCoffee.orchestrate(
                Cart.CREATE, CreateCartTemplate.SIMPLE_USD, cartValidations,
                Cart.GET, firstGetCartValidations,
                Cart.GET_EPM,
                Cart.UPDATE, PatchCartTemplate.SIMPLE, patchCartValidations,
                Cart.GET_FO, gfoValidations,
                Payment.CREATE, CreatePaymentTemplate.REQUEST_WITH_FO_ID, payValidations,
                Sale.GET,
                Payment.GET, getPayValidations,
                Cart.GET, secondGetCartValidations,
                ClientChannel.ProxMemberRememberMe, buyer, facilitator, seller);

    }
```
Go ahead and run this. You should see ~160 validations get triggered. If we care to validate the GET/Sale endpoint, we'd probably have more assertions. If you're feeling adventurous, feel free to bolt on a dataProvider and supply a number of different templates. You can also supply a varying number of funding instruments and other such details. For now, let's settle for something simple:

```java
    @DataProvider(name = "miscCartAndPatchTemplates")
    public Object[][] filterCoffeeDP() {
        Object[][] obj = {
                {CreateCartTemplate.SIMPLE_USD, PatchCartTemplate.SIMPLE},
                {CreateCartTemplate.SIMPLE_EUR, PatchCartTemplate.SIMPLE},
                {CreateCartTemplate.MSP_2PUS, PatchCartTemplate.MSP_ADD_PU}
                // so on and so forth
        };
        return obj;
    }
    
    @Test(dataProvider = "miscCartAndPatchTemplates")
    public void simpleProxEndToEndTest(CreateCartTemplate createCartTemplate, PatchCartTemplate patchCartTemplate) throws Exception {

        PayPalUser buyer = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.BUSINESS, Country.DE,
                Info.Euro, null, CreditCardType.VISA);
        PayPalUser facilitator = UserBaseUtil.getLegacyUser(masterProxFacilitatorEmail);
        PayPalUser seller = UserBaseUtil.getLegacyUser(masterDESellerEmail);

        List<ValidationUnit> cartValidations = validationUtility.build(Cart.CREATE, createCartTemplate);
        List<ValidationUnit> firstGetCartValidations = cartValidations;
        List<ValidationUnit> patchCartValidations = validationUtility.build(Cart.UPDATE, createCartTemplate, patchCartTemplate);
        List<ValidationUnit> gfoValidations = validationUtility.build(Cart.GET_FO,
                GetFundingOptionsValidationTemplate.DEF_CC_WITHOUT_ALT, createCartTemplate, patchCartTemplate);
        List<ValidationUnit> payValidations = validationUtility.build(Payment.CREATE, createCartTemplate,
                patchCartTemplate, PaymentValidationTemplate.INSTANT_SALE);
        List<ValidationUnit> getPayValidations = validationUtility.build(Payment.GET, createCartTemplate,
                patchCartTemplate, GetPayValidationTemplate.PAYEE_EMAIL);
        List<ValidationUnit> secondGetCartValidations = patchCartValidations;

        filterCoffee.orchestrate(
                Cart.CREATE, createCartTemplate, cartValidations,
                Cart.GET, firstGetCartValidations,
                Cart.GET_EPM,
                Cart.UPDATE, patchCartTemplate, patchCartValidations,
                Cart.GET_FO, gfoValidations,
                Payment.CREATE, CreatePaymentTemplate.REQUEST_WITH_FO_ID, payValidations,
                Sale.GET,
                Payment.GET, getPayValidations,
                Cart.GET, secondGetCartValidations,
                ClientChannel.ProxMemberRememberMe, buyer, facilitator, seller);

    }
```
A piece of trivia: You've just implemented what used to be a P0 smoke test in the old suite (the only difference being that the buyer has both a CC and a bank, which are very trivial for us to add). [Here's the old test](https://github.paypal.com/Checkout-Playground/SymphonyFunctionalTest/blob/master/SymphonyBluefinTests/src/test/java/com/paypal/test/symphony/test/tdd/prox/member/GetFundingOptionsTests.java#L250) sitting at a whopping 170 lines. Our test is sitting at a cool 25 lines, and that's not including how we've dataProvisioned so many inputs which in the old suite simply didn't happen.

Perhaps this is a good time to pause and reflect on the learnings thus far. We've discovered how to create users in a neat and efficient manner, payloadTemplates to capture requests, validationTemplates to capture situations, the powerful validationUtility to handle the grunt work of building long lists, along with the simplicity of orchestrating additional APIs that almost always just added 1 single line of code. The question to ask yourself is, is this readable? Can it be extended? And now that you're equipped with this knowledge, is writing a prox based test case easy? We're just beginning to scratch the surface of what's possible with filterCoffee. In the next exercise, we'll explore how to add code in filterCoffee that doesn't currently exist. Hope you've enjoyed the journey so far!

### Adding a new payload template
Let's say you were approached about a bug in LIVE, and you diligently went ahead and pulled all the CAL logs. It's a prox transaction, and you capture the createCart payload that eBay sends us. Now you'd like to write a test orchestrating a number of APIs, but for starters you want to use this brand new payload that you've extracted. What are your next steps?

Simple, go to the CartPayloadTemplates class (because that's where all the cart-based payload templates live) and look at the CreateCartTemplate enum. Add an enum there, for example, EBAY_CART_REQUEST_NSFW. Now you need to provide a path where your json file lives. Go ahead and add the file at say "prox/prox-create-cart-request-ebay-nsfw.json". Your enum should look something like:

```java
    public enum CreateCartTemplate implements PayloadTemplate {

        EBAY_CART_REQUEST_NSFW("prox/prox-create-cart-request-ebay-nsfw.json"),
        SIMPLE_USD("prox/prox-create-cart-request-USD.json"),
        ...
    }    
```
Once you've done this, everyone will access to it while calling the Cart.CREATE endpoint. To verify what is actually happening under the hood, goto the [CartServiceImpl](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/impl/CartServiceImpl.java). There you'll find a method called the createCart which [internally invokes](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/impl/CartServiceImpl.java#L95) a cartRequestMapper method. Navigate [into this method](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/mapper/CartServiceRequestMapper.java#L132). Here you'll notice that all we're doing is using the enum to arrive at the file location. So your work is done.

### Adding a new field validation
You're reviewing some of the existing payload templates in Cart, and you closely review the CreateCartTemplate.SIMPLE_USD payload. You notice there's a field in there called "description". That's odd, you didn't notice this getting validated in your end to end test. You decide you'd like to change that and add it in such a way that no matter who uses the validationUtility to build the createCart validations, they're all able to benefit from your changes. You begin by surveying the validationUtility by navigating into its build method. You end up at [this method](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/util/ValidationUtil.java#L50) that is supposed to build the list on your behalf. While trying to step into the buildValidationUnits method, you notice there's a bunch of classes that implement it. Being the clever engineer that you are, you pick the CartValidationBuilder because that feels like the relevant builder. You keep stepping through the code and finally arrive at the [buildListForCreateCartUsingJson method](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/validation/builder/helper/CartValidationBuilderHelper.java#L60) and see that it delegates the work to the [buildValidationUnitsForJsonTree method](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/validation/builder/helper/CartValidationBuilderHelper.java#L361). You know that the "description" field lives in the purchaseUnit, so you navigate to the [getPurchaseUnitValidations method](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/validation/builder/helper/CartValidationBuilderHelper.java#L401). Now we know where we need to appropriately parse the json file and add the validation.

Before we are in a hurry to build a validationUnit for the description, let's recall that in order to ever build a list, we need to have a relevant field. Is there a "description" field in the CartResourceFields? If not go ahead and add it. 

```java
public enum CartResourceFields implements FieldAlias {

    DESCRIPTION("purchase_units[x]/description"),
    // skipping rest of the code
    
}    
```

Now that you've added it, your validation construction code may look something like:

```java
public class CartValidationBuilderHelper {
    
    ...
    
    private List<ValidationUnit> getPurchaseUnitValidations(JsonNode jsonValue) {
    
        ... // skipping pre-existing code for brevity
        
        // Get description validation
        List<ValidationUnit> descriptionValidation = getDescriptionValidation(purchaseUnit, puIndex, referenceId);
        if (CollectionUtils.isNotEmpty(descriptionValidation)) {
            validationUnits.addAll(descriptionValidation);
        }
    }
    
    private List<ValidationUnit> getDescriptionValidation(JsonNode purchaseUnit, int puIndex, String referenceId) {
        
        if (!ValidationBuilder.elementExists(purchaseUnit, "description")) {
            return null;
        }
        
        List<ValidationUnit> validationUnits = new ArrayList<ValidationUnit>();
        String descriptionText = purchaseUnit.get("description").asText();
        CartField cartField = getCartField(DESCRIPTION, puIndex, referenceId);
        ValidationBuilder.addValidation(cartField, descriptionText, validationUnits);
        return validationUnits;
    }
```
After this we can feel confident that this code will fish out any description and add it to the list of validations. But, we need the appropriate validator to "look" for this particular field and perform the appropriate validation. Since this is for CreateCart, we'll navigate to the [CartServiceResponseValidator](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/validator/CartServiceResponseValidator.java) which holds the [validateCartResource](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/validator/CartServiceResponseValidator.java#L100) method. Here you'll notice a number of switch cases that perform some action for validation. All we have to do is add a case statement so that we can inspect the returned cart resource and check the description. The code might look something like:

```java
public class CartServiceResponseValidator extends MapperBase {
   
    ...
    
    private void validateCartResource(int statusCode, JsonNode jsonResponse, List<ValidationUnit> validationUnitList,
            String msgPrefix) {
       
       switch (fieldKey) {
       // skipping code for brevity
       
        case DESCRIPTION:
        actualValue = cart.getPurchaseUnits().get(cartField.getPuIndex()).getDescription();
        simpleAssert(actualValue, vu.getExpectedValue(), cartField, msgPrefix);
        break;
        
       }     
    }        
```
Notice how the exact purchaseUnit index comes in handy especially in an MSP use-case where the right description is picked up. Once you've done this, you've enabled description-based validation for every single test that uses the standard utility to build validations for the createCart api.

### Testing your WOWO based feature
So you're working on this interesting feature but it is guarded in the symphony code with a WOWO (let's assume `WOWO_AWESOME_NEW_FEATURE`). This code may be in your fork or in develop, doesn't matter; you're just interested in testing it when the WOWO is enabled. The standard way to do this is navigate to the `paymentapiplatformserv.txt` file on the stage, tweak the WOWO to `true`, restart the service. Once restarted, you're going to run your functional tests against that particular stage and see if you get the desired result. In filterCoffee, you can skip this manual context switching by automating the WOWO flip from the test code itself. There are 2 flavors of doing this - flipping the WOWO globally for a test class, and flipping the WOWO locally for a particular transaction and then "resetting" it. Let's tackle the first case.

You will find a [WOWOUtil](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/util/WOWOUtil.java) class in the `com.paypal.test.filtercoffee.util` package. You notice that it has a static method called [changeWOWO](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/util/WOWOUtil.java#L28) which handles the flipping of the WOWO based on a [WOWOTemplate](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/payload/WOWOPayloadTemplates.java#L5). You go to this enum and see a number of other such WOWOs. These have been modeled after the actual WOWO names in the paymentapiplatformserv.txt file, so you go ahead and add a field called WOWO_AWESOME_NEW_FEATURE. Once you're done with this, you go back to the test you're writing and add the following:

```java
public class FilterCoffeeTests {

    @BeforeClass
    public void enableWOWO() {
        WOWOUtil.changeWOWO(WOWOTemplate.WOWO_AWESOME_NEW_FEATURE, true);
    }

    @BeforeClass
    public void disableWOWO() {
        WOWOUtil.changeWOWO(WOWOTemplate.WOWO_AWESOME_NEW_FEATURE, false);
    }
    
    @Test
    public void yourTestForTheAwesomeNewFeature() {
        // ...
    }
}
```
With this you can be sure that at the end of your test case, the "reset" of the WOWO is taken care of so that the environment isn't tampered for someone else's test cases that may run later on. But what if you wanted to flip a WOWO mid-way through a transaction? Somehow you're convinced that this is an important use-case to test symphony's behavior during a rollout where a POST/Payment call may arrive at a box that has the WOWO enabled, but then another exact same POST/Payment call may arrive at another box that doesn't have the "fresh" build (and its WOWO is disabled). Granted, this is a pretty contrived example, but being the diligent engineer you are you want to make sure everything is completely bulletproof. It's obvious that we need a mechanism of invoking this WOWOUtil in the orchestration method, but is there such an abstraction that we can take advantage of?

Fortunately, the WOWOUtil has been "wrapped" as a [WOWO](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/WOWO.java) endpoint as well. Now technically, a WOWO isn't a restful endpoint or an ASF endpoint; it's just some config file that we'd like to manipulate. FilterCoffee considers all such flavors of actions as generic "endpoints"; these may be specific calls to a message queue, caching layer, config file, CDB setting, RESTful api etc. The takeaway is that endpoints of all types are first class citizens and help in automation. Let's look at what a WOWO flip baked into an orchestration looks like:

```java
    @Test
    public void yourTestForTheAwesomeNewFeature() {

        // assume users have been created
        
        filterCoffee.orchestrate(
                // start the test with the WOWO enabled
                WOWO.ENABLE, WOWOTemplate.WOWO_AWESOME_NEW_FEATURE,
                Cart.CREATE, CreateCartTemplate.SIMPLE_USD,
                Cart.GET_FO,
                Payment.CREATE, CreatePaymentTemplate.REQUEST_WITH_FO_ID,
                // disable the WOWO mid-way
                WOWO.DISABLE, WOWOTemplate.WOWO_AWESOME_NEW_FEATURE,
                Payment.CREATE, CreatePaymentTemplate.REQUEST_WITH_FO_ID,
                ClientChannel.ProxMemberRememberMe, buyer, facilitator, seller);
    }
```
Pretty simple stuff. If you dig a little deeper into the WOWO and its corresponding WOWOServiceImpl, you'll see that it actually delegates work to the WOWOUtil internally so that there's maximum code-reuse.

## Validating the Mayfly session
Let's say you're working on a feature where certain flags get set in mayfly as a result of an api call (ex: createCheckoutSession), but those fields are not echoed back in the api response. How can you best validate whether the intended state changes were made? Back in the day, we could tail the logs on the stage, move them to a flat file, search for the mayfly session and then manually inspect the field of interest. Seems like a huge overhead for something so fundamental. Instead of all this, you can go ahead and use the [Mayfly endpoint](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/Mayfly.java) which will dump the mayfly session automatically at any point in the orchestration. Let's tackle this in 2 parts. Step1: Simplify getting the mayfly session so that we can inspect it manually. Step2: automate the inspection of the mayfly response like a "validation".

```java
    @Test
    public void sneakyFeature() {

        // assume users have been created
        
        filterCoffee.orchestrate(
                ExpressCheckout.SET, SetECTemplate.SIMPLE_SALE_USD,
                CheckoutSession.CREATE,
                Mayfly.GET,
                ClientChannel.AriesWebMember, buyer, seller);
    }
```
You will notice in your console that the whole mayfly session is actually printed. This definitely beat sshing tailing and parsing manually. For the purposes of illustration, let's assume that you were deeply interested in the planning route that helped generate plans and ultimately the checkoutSession resource. Now, you know that most of the routing related logic lives in the [PaymentsPathServiceImpl](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/app-core/src/main/java/com/paypal/platform/payments/core/service/impl/PaymentsPathServiceImpl.java#L102). You notice that the field lives in the purchaseDataVO as a `planningDecision`. Great, now we know how we can start parsing our response.

You goto the MayflyServiceImpl and look at the getSession method. You notice towards the end that there is an invocation of the MayflyServiceResponseValidator's [validateGetSession method](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/validator/MayflyServiceResponseValidator.java#L24). You then see that there is already a fieldKey called `PLANNING_ROUTE` that is handled for us. This means all you have to do is write an appropriate validationUnit which should automate the inspection of the planningRoute. Now you think to yourself "Someone already wrote a validator for mayfly, surely they were diligent enough to write a validationBuilder accessible via the validationUtility. Perhaps I can tap into that?". Cautious of making assumptions, you click on the Mayfly enum to inspect its contents and much to your dismay, you notice that [there is NO corresponding validationBuilder](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/Mayfly.java#L29).

Now, maybe you're in a hurry and you don't want to invest the time in building a full fledged mayflyValidationBuilder, so instead you decide to still tap into the validationUtility but without requiring "custom" endpoint template based validations. Essentially, you can use the validationUtility to simply "zip" a list for you. You end up writing something like:

```java
    @Test
    public void sneakyFeature() {

        // assume users have been created
        
        // Notice that you're not leading with the "relevant" endpoint like MAYFLY.GET
        // Instead this build method simply "zips" the list into the validationUnit pojos.
        List<ValidationUnit> mayflyValidations = validationUtility.build(
                SessionVarDataFields.PLANNING_ROUTE, PaymentsDecisionTypeEnum.PAYMENTS_20);
        
        filterCoffee.orchestrate(
                ExpressCheckout.SET, SetECTemplate.SIMPLE_SALE_USD,
                CheckoutSession.CREATE,
                Mayfly.GET, mayflyValidations,
                ClientChannel.AriesWebMember, buyer, seller);
    }
```
## Introducing a new endpoint
Let's say you're working on a new product that requires talking to [PayPal's billingPlans apis](https://developer.paypal.com/docs/api/#billing-plans-and-agreements). You scan around the `com.paypal.test.filtercoffee.endpoint.resource` package and find nothing of the sort. So you decide to go ahead and build out the endpoint. Naturally you're interested in building this in such a way that it is accessible for anyone else writing a test in the future. So you pause and meditate on all the different things that you need to put to together.

* An endpoint alias. We've seen things like Cart.CREATE; it would be nice if we could have an abstraction called BillingPlans.CREATE that would underneath delegate the creation of billingPlans. We can call the enum BillingPlans.
* An endpoint service. Naturally, there needs to be a service that actually makes the api call. You decide to do the professional thing and mantain an interface that will list out all the different billingPlan related APIs. This way, whichever class attempts to implement the functionality, it will always be aware of the other APIs. We can call it BillingPlansService. 
* An endpoint serviceImpl. Some entity needs to have an intimate knowledge of how to build the request, understand what port to send the request to and what the required headers are to receive a successful response. All this logic needs to be invoked in one central place, and you believe that the serviceImpl is a good place to start. Let's call this the BillingPlansServiceImpl.
* A service request mapper. As mentioned previously, you foresee many use-cases having their own quirks of building the request, so you decide to park this functionality in a separate request mapper. Perhaps BillingPlanServiceRequestMapper is an obvious name.
* A service response validator. Since we're trying to test things, we should park all our validations in one class. Let's call this one the BillingPlansServiceResponseValidator. A little verbose no doubt, but there's no ambiguity when you read it.

So far throughout this tutorial, we have definitely seen each one of the above flavors of classes. The design is intuitive and closely mirrors what actually lives in the main symphony code-base. Let's see how successful we can be with implementing this from scratch. We start with the endpoint alias.

```java
package com.paypal.test.filtercoffee.endpoint.resource;

import com.paypal.test.filtercoffee.service.EndpointService;
import com.paypal.test.filtercoffee.validation.builder.ValidationBuilder;

public enum BillingPlans implements Endpoint {
    CREATE,
    UPDATE,
    GET,
    LIST;

    @Override
    public EndpointService getEndpointService() {
        // Oh no! Where's our service?
        return null;
    }

    @Override
    public ValidationBuilder getValidationBuilder() {
        // We are lazy so we will skip this for now
        return null;
    }

}

```
This looks good, but we would like to color this with some more information. So we'll add an endpointName field and has a textual description for each of the APIs.

```java
package com.paypal.test.filtercoffee.endpoint.resource;

import com.paypal.test.filtercoffee.service.EndpointService;
import com.paypal.test.filtercoffee.validation.builder.ValidationBuilder;

public enum BillingPlans implements Endpoint {
    CREATE("createBillingPlan"),
    UPDATE("updateBillingPlan"),
    GET("getBillingPlan"),
    LIST("listBillingPlan");

    private String endpointName;

    BillingPlans(String endpointName) {
        this.endpointName = endpointName;
    }
    
    // This makes debugging in the IDE easier
    @Override
    public String toString() {
        return endpointName;
    }

    @Override
    public EndpointService getEndpointService() {
        // Oh no! Where's our service?
        return null;
    }

    @Override
    public ValidationBuilder getValidationBuilder() {
        // We are lazy so we will skip this for now
        return null;
    }

}

```
Cool. Looks like the next step is to "wire in" the serviceImpl that is clearly missing from this enum's getEndpointService() method. Let's do that, but first start with building up the interface.

```java
package com.paypal.test.filtercoffee.service;

import java.util.concurrent.ConcurrentHashMap;

import com.paypal.test.filtercoffee.endpoint.resource.Endpoint;
import com.paypal.test.filtercoffee.source.ExecutionUnit;

public interface BillingPlansService {

    // The execution unit stores info about the parties involved, the payload, api call etc
    // The orchestrationContext stores transaction context in case we need to reach into a previous
    // api's response. For ex: GET/billing-plan would reach into POST/billing-plan response for the plan-id
    Object createBillingPlan(final ExecutionUnit execUnit, final OrchestrationContext orchestrationContext)
            throws Exception;

    Object updateBillingPlan(final ExecutionUnit execUnit, final OrchestrationContext orchestrationContext)
            throws Exception;
    
    Object getBillingPlan(final ExecutionUnit execUnit, final OrchestrationContext orchestrationContext)
            throws Exception;
    
    Object listBillingPlans(final ExecutionUnit execUnit, final OrchestrationContext orchestrationContext)
            throws Exception;
}

```
Now we can start writing the BillingPlansServiceImpl.

```java
package com.paypal.test.filtercoffee.service.impl;

import java.util.concurrent.ConcurrentHashMap;

import com.paypal.test.filtercoffee.endpoint.resource.Endpoint;
import com.paypal.test.filtercoffee.service.BillingPlansService;
import com.paypal.test.filtercoffee.service.EndpointService;
import com.paypal.test.filtercoffee.source.ExecutionUnit;

public class BillingPlansServiceImpl implements EndpointService, BillingPlansService {

    @Override
    public Object execute(ExecutionUnit execUnit, OrchestrationContext orchestrationContext) throws Exception {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public Object createBillingPlan(ExecutionUnit execUnit, OrchestrationContext orchestrationContext)
            throws Exception {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public Object updateBillingPlan(ExecutionUnit execUnit, OrchestrationContext orchestrationContext)
            throws Exception {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public Object getBillingPlan(ExecutionUnit execUnit, OrchestrationContext orchestrationContext)
            throws Exception {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public Object listBillingPlans(ExecutionUnit execUnit, OrchestrationContext orchestrationContext)
            throws Exception {
        // TODO Auto-generated method stub
        return null;
    }

}

```
Cool looks like we've got our serviceImpl even though it doesn't do anything particularly useful right now. Go back to the BillingPlans enum and "wire in" this new serviceImpl. So your code should look like:

```java
    @Override
    public EndpointService getEndpointService() {
        // Here's our service!
        return new BillingPlansServiceImpl();
    }
```
Let's now focus on having the "execute" method delegate to the right api method underneath. We'll go ahead and follow a simple pattern the other serviceImpls do:

```java
    @Override
    public Object execute(ExecutionUnit execUnit, OrchestrationContext orchestrationContext) throws Exception {
        
        BillingPlans billingPlanApi = (BillingPlans) execUnit.getApiUnit().getApiMethod();
        switch (billingPlanApi) {
        case CREATE:
            return createBillingPlan(execUnit, orchestrationContext);
        case GET:
            return getBillingPlan(execUnit, orchestrationContext);
        case LIST:
            return listBillingPlans(execUnit, orchestrationContext);
        case UPDATE:
            return updateBillingPlan(execUnit, orchestrationContext);
        default:
            throw new IllegalArgumentException("Unknown BillingPlans endpoint called.");
        }
    }
```
Cool, now we don't have to worry about the execute method delegating to the right method. Now let's focus our attention on the first api call which is the createBillingPlan api. After [surveying the PayPal REST Api docs](https://developer.paypal.com/docs/api/#create-a-plan) you'll find that there's a payload that's required in the request. Since we're trying to simplify the transmission of the payload from the test to the serviceImpl, let's quickly create a payload template that handles this for us. We'll create a class called BillingPlansPayloadTemplates and host an enum called CreateBillingPlanTemplate. Here we'll create a simple alias field (let's call it `SIMPLE_REQ`) and point it to the actual json payload for now. Since we're interested in just getting something to work quickly, we'll copy the payload from the docs and park them in some folder temporarily.

```java
package com.paypal.test.filtercoffee.template.payload;

public class BillingPlansPayloadTemplates {

    public enum CreateBillingPlanTemplate implements PayloadTemplate {

        SIMPLE_REQ("prox/simple_create_billing_request.json");

        private String filePath;

        public String getFilePath() {
            return this.filePath;
        }

        CreateBillingPlanTemplate(String filePath) {
            this.filePath = filePath;
        }
    }

}
```
This file below needs to be parked in the src/test/resources under the prox folder.
```json
{
    "name": "T-Shirt of the Month Club Plan",
    "description": "Template creation.",
    "type": "fixed",
    "payment_definitions": [
        {
            "name": "Regular Payments",
            "type": "REGULAR",
            "frequency": "MONTH",
            "frequency_interval": "2",
            "amount": {
                "value": "100",
                "currency": "USD"
            },
            "cycles": "12",
            "charge_models": [
                {
                    "type": "SHIPPING",
                    "amount": {
                        "value": "10",
                        "currency": "USD"
                    }
                },
                {
                    "type": "TAX",
                    "amount": {
                        "value": "12",
                        "currency": "USD"
                    }
                }
            ]
        }
    ],
    "merchant_preferences": {
        "setup_fee": {
            "value": "1",
            "currency": "USD"
        },
        "return_url": "http://www.return.com",
        "cancel_url": "http://www.cancel.com",
        "auto_bill_amount": "YES",
        "initial_fail_amount_action": "CONTINUE",
        "max_fail_attempts": "0"
    }
}
```
Now the expectation is that the test will pass this payload which the serviceImpl will receive as a payloadTemplate embedded in the executionUnit. This should solve the problem of "acquiring" the request payload. If you examined headers used in the CURL command in the REST api docs, you'll notice it requires the Authorization Bearer <AccessToken>. This means we need to wire in some oAuth apis that will fetch the appropriate accessToken on our behalf. Finally, it's clear that the port is going to be that of the facade which will then delegate it to the billing-plans service. This port has been known to be 11888.

We can take our cue from the CartServiceImpl which hosts the [createCart method](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/ec30a41bb5393423272a7b080569afc886d8619e/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/impl/CartServiceImpl.java#L98). The createCart method too goes through some rigorous auth based and developer application api calls internally. Since we're feeling silly and in a hurry, let's just go ahead and copy that code over here:

```java
public class BillingPlansServiceImpl implements EndpointService, BillingPlansService {

    private final ProxServiceBase proxServiceBase = new ProxServiceBase();
    private final String BILLING_PLANS_URI = "/v1/payments/billing-plans";

    @Override
    public Object createBillingPlan(ExecutionUnit execUnit, OrchestrationContext orchestrationContext)
            throws Exception {

        // Extract the facilitator who is making the api call in this case
        String facilitatorAccountNumber = execUnit.getApiUnit().getFacilitator().getAccountNumber();
        // Register the faciltator as an application and get the accessToken
        // This step happens just once when the facilitator is first onboarded. proxServiceBase takes care of caching.
        proxServiceBase.performAuthCeremony(facilitatorAccountNumber);
        // We get our accessToken called "brandNewCreds"
        String brandNewCreds = proxServiceBase.renewAccessToken(facilitatorAccountNumber);

        // Let's meditate on what else needs to be done for now. 
        return null;
    }
```
Now we need to wire in a request mapper that should extract the payloadTemplate and load the json file appropriately. Let's build a simple request mapper:

```java
package com.paypal.test.filtercoffee.service.mapper;

import java.io.IOException;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.paypal.test.filtercoffee.source.ExecutionUnit;
import com.paypal.test.filtercoffee.template.payload.BillingPlansPayloadTemplates.CreateBillingPlanTemplate;

public class BillingPlansServiceRequestMapper extends MapperBase {

    public JsonNode mapCreateBillingPlanRequest(ExecutionUnit execUnit) throws JsonProcessingException, IOException {

        CreateBillingPlanTemplate templ = (CreateBillingPlanTemplate) execUnit.getApiUnit().getPayloadTemplate();
        ObjectNode jsonRequest = loadJsonFile(RESOURCE_PATH, templ.getFilePath());
        return (JsonNode) jsonRequest;
    }

}
```
Since we're making standard REST api calls (POST/GET etc) it's important to go through a standard abstraction that hosts this for us. In filterCoffee, we have the notion of a ServiceBridge that hosts all these different API calls. So to tap into this, all we have to do is extend the ServiceBridge.

```java
public class BillingPlansServiceImpl extends ServiceBridge implements EndpointService, BillingPlansService {

    private final ProxServiceBase proxServiceBase = new ProxServiceBase();
    private final String BILLING_PLANS_URI = "/v1/payments/billing-plans";
    private BillingPlansServiceRequestMapper billingRequestMapper = new BillingPlansServiceRequestMapper();

    @Override
    public Object createBillingPlan(ExecutionUnit execUnit, OrchestrationContext orchestrationContext)
            throws Exception {

        //Extract the request
        JsonNode request = billingRequestMapper.mapCreateBillingPlanRequest(execUnit);

        // Extract the facilitator who is making the api call in this case
        String facilitatorAccountNumber = execUnit.getApiUnit().getFacilitator().getAccountNumber();
        // Register the faciltator as an application and get the accessToken
        // This step happens just once when the facilitator is first onboarded. proxServiceBase takes care of caching.
        proxServiceBase.performAuthCeremony(facilitatorAccountNumber);
        // We get our accessToken called "brandNewCreds"
        String brandNewCreds = proxServiceBase.renewAccessToken(facilitatorAccountNumber);

        ServiceBridgeUnit serviceBridgeUnit = buildServiceBridgeUnit(request, BILLING_PLANS_URI, null,
                facilitatorAccountNumber, ServiceKey.ProxPaymentsService, brandNewCreds, null);
        Pair<JsonNode, StatusType> responsePair = post(serviceBridgeUnit);
        JsonNode jsonResponse = responsePair.getLeft();
        return jsonResponse;
    }
```
Now we'll put together a very basic test calling only a single endpoint:

```java
    @Test
    public void basicBillingPlanTest() throws Exception {

        PayPalUser buyer = PPUserFactory.createDummyPayPalUser();
        PayPalUser facilitator = UserBaseUtil.getLegacyUser(masterProxFacilitatorEmail);

        filterCoffee.orchestrate(
                BillingPlans.CREATE, CreateBillingPlanTemplate.SIMPLE_REQ,
                ClientChannel.ProxMemberRememberMe, buyer, facilitator);
    }
```
If you run the test and see the final response, you will notice it emits a 403 statusCode REQUIRED_SCOPE_MISSING which means that we were a little flippant about the oauth orchestration. More work needs to be done to understand what kinds of oauth APIs need to be called underneath, but at a high level, this should have given you an idea of how to assemble a service that is accessible for orchestration.

[TODO: Resolve the oauth api, and add a response validator. Next, wire in the GET/billing-plans api to extract the planId from the old createBillingPlans response and make the api call]
[TODO: Walk through the ServiceBridge and JsonClientFactory]

## Analyzing different flavors of tests

Let's take a look at a different mix of tests that currently exist in filterCoffee and evaluate whether it is easy or hard to understand the tests. We'll try to sweep through a number of different use-cases and see what special provisions are made to tackle them effectively.

```java
    @Test(description = "ExpiredCC - Symphony", suiteName = "AriesP1Tests", priority = 0, testName = "Create XOSession with buyer having expired CC")
    public void testBuyerWithExpiredCC() throws Exception {

        PayPalUser buyer = UserBaseUtil.createBuyerWithExpiredCCWithoutRecycling(PPAccountType.PERSONAL, Country.US, Info.US_Dollar,
                CreditCardType.VISA);
        PayPalUser seller = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.BUSINESS, Country.US,
                Info.US_Dollar, UserAccountFlags.WUSER_FLAG_HAS_CONFIRMED_EMAIL, CreditCardType.VISA);

        List<ValidationUnit> createXOSessionValidations = validationUtility.build(CheckoutSession.CREATE,
                CheckoutSessionContingencyTemplate.EXPIRED_PAYMENT_CARD);

        filterCoffee.orchestrate(
                ExpressCheckout.SET, SetECTemplate.SIMPLE_SALE_USD,
                CheckoutSession.CREATE, createXOSessionValidations,
                Wallet.ADD_CARD, AddCardTemplate.VISA,
                FundingOptions.GENERATE, GenerateFOTemplate.NEWLY_ADDED_CARD,
                FundingOptions.GET,
                CheckoutSession.APPROVE,
                ExpressCheckout.GET,
                ExpressCheckout.DO, DoECTemplate.SIMPLE_SALE_USD,
                ClientChannel.AriesWebMember, buyer, seller);
    }
```
Let's take a moment to understand this test. Seems like the test is trying to do something with a buyer that has an expired CreditCard (from the name). At first glance, it appears that there's a method invocation that builds a user with an expired CC without recycling. If you step inside that method, you will notice that there's a pojo being set strategically `buyerSettings.setExpireAllCCsInWallet(true)`. If you check the call heirarchy (Cmd+Shift+G) on the correponding getter (i.e. `buyerSettings.getExpireAllCCsInWallet()`) you will find that it is invoked in the UserUtility and a [BuyerSettingUtility method expires all cards](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/user/util/BuyerSettingsUtility.java#L17) in the wallet via a DB call. We will address why the method includes the "withoutRecycling" part later on.

Seller construction looks pretty standard, so nothing fancy happening there. Let's move onto the next line that constrcuts a list of validations for createXOSession. This looks interesting. There seems to be a [CheckoutSessionContingencyTemplate](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/validation/CheckoutSessionValidationTemplates.java#L11) that is being used here. On inspecting the enum you can see that it is a validationTemplate. Next we'd like to see what this little enum actually maps to in terms of a validationUnit list. Hit Cmd+Shift+G to see where the `CheckoutSessionContingencyTemplate.EXPIRED_PAYMENT_CARD` is being used. You'll find that it is being used [in this method](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/validation/builder/helper/CheckoutSessionValidationBuilderHelper.java#L201) deep inside the CheckoutSessionValidationBuilderHelper.

Taking a closer look at the list, it appears that the following fields ought to be set: PAYMENT_CONTINGENCY_CAUSE_NAME, POTENTIAL_RESOLUTION_NAME, STATUS_CODE, PAYMENT_APPROVED, CHECKOUT_SESSION_STATE, and PAYMENT_CONTINGENCY_NAME. Now bear in mind that this could have been manually constructed in the test, but the author of the test decided to plug this in the validation builder so that it is easily accessible to anyone else who may be testing a similar scenario elsewhere. One thing stands out though: there's 2 instances of `POTENTIAL_RESOLUTION_NAME` that are being used in the list. At first glance that seems odd, but here is the reasoning: There is no guarantee of ordering when it comes to resolution names. Partially this is due to the fact that the underlying services that report such errors do not make any such guaranteers, but largely this is due to the fact that symphony could end up talking to completely different services altogether (Checkoutserv, MPS native, Mozart1.0 and Mozart2.0). To handle this uncertainty regarding the ordering of the resolutionNames, the author of the test thought it best that what needs to be tested is the __presence__ of the 2 resolution names. How can we make this claim confidently? The simplest way to check is by opening the [CheckoutSessionServiceResponseValidator](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/validator/CheckoutSessionServiceResponseValidator.java#L39) and inspect the way this field is being validated. [You will find](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/validator/CheckoutSessionServiceResponseValidator.java#L148) that there's a simple loop checks if the resolution-name exists anywhere in the list. Cool, so this demonstrates that the tester can have flexibility in having "softer" assertions where you check for the "presence of something somewhere" instead of a more strict "presence of something in one particular place indexed exactly". Cool, so now we're able to reconcile all the steps prior to calling the `orchestrate` method.

Next let's inspect the orchestration. There's a SetEC with a particular payload. Once again, if you're curious about what `SIMPLE_SALE_USD` means exactly, click on the enum and see what you find. Alas, there's no json file to look at! Don't worry, this is by design. Clearly, some entity in filterCoffee understands this enum in a particular way. Putting your architect hat back on, you figure "Hmm, if I were designing this, I would imagine there's a serviceImpl that's handling the setEC call. That serviceImpl is probably delegating to some requestMapper to construct the request on its behalf." This would be an astute thought, and equipped with this theory you decide to pull up the [ExpressCheckoutServiceReqestMapper](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/mapper/ExpressCheckoutServiceRequestMapper.java). There, you will find our beloved enum in a case statement in the [mapSetECRequest method](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/mapper/ExpressCheckoutServiceRequestMapper.java#L46). You will notice it delegates this construction to yet another class called the [ExpressCheckoutRequestFactory](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/mapper/ExpressCheckoutRequestFactory.java#L26). You see that it sets an intent of SALE along with some other parameters.

Next, you see a CheckoutSession.CREATE and you suspect that the response will send back some contingency. The test then calls a Wallet.ADD_CARD which looks interesting. You decide to look a little deeper into it by clicking on the Wallet enum and finding its [corresponding endpointService](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/endpoint/resource/Wallet.java#L30). Once again, you navigate to the WalletServiceImpl and find how it is [creating the walletRequest and making the POST call](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/impl/WalletServiceImpl.java#L49). It's clear that here it talking to a RESTful api at a particular endpoint. Sidenote: If filterCoffee were talking to the ASF wallet service, and as a part of the migration we needed to talk to another flavor of service (like REST), note that the change could be localized to just the serviceImpl without breaking any other existing test cases. You smile at the separation of concerns, and then move on to the remaining orchestration logic.

There's a FundingOptions.GENERATE call with a peculiar template called `GenerateFOTemplate.NEWLY_ADDED_CARD`. Before clicking through more code, you try to silently hazard a guess as to what is happening here. You think to yourself "Hmm, it's clear that to make this API call, we would need the exact fundingId as a query parameter in the endpoint. But the fundingId is something we could have gotten from the Wallet.ADD_CARD api. Somehow the FundingOptionsServiceImpl is getting the fundingId from the Wallet.ADD_CARD response. My guess is that it's that sneaky orchestrationContext that is helping the FundingOptionsServiceImpl in supplying the fundingOptionId surreptitiously so that it can add it to the endpoint uri".

If you thought the above, then [you're absolutely right](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/impl/FundingOptionsServiceImpl.java#L137). This is in fact the way CheckoutSession.CREATE gets its payload (the cart_id) as well.

Next up is a FundingOptions.GET api call that doesn't seem to validate anything. Then a CheckoutSession.APPROVE, then ExpressCheckout.GET, and finally an ExpressCheckout.DO with its payloadTemplate. Similar to the SetEC case, you suspect that there's some requestMapper that understands the DoECTemplate.SIMPLE_SALE_USD. Finally you notice that the client is AriesWebMember.

Let's take a look at another example from devplat. Here's a [futurePayments test case](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/test/devplat/futurepayments/FuturePaymentsSaleP1Tests.java#L33).

```java
    @Test(dataProvider = "BuyerWithVaryingFundingsAndCountries", description = "Happy Path - Symphony", suiteName = "FuturePaymentsSaleP1Tests", priority = 0, testName = "e2eSaleForBuyerWithDifferentBasicFundingInstruments")
    public void e2eSaleForBuyerWithDifferentBasicFundingInstruments(Country country, Info currency, Object fundingInstrumentData) throws Exception {

        PayPalUser seller = PayPalAccountHelper.loadLegacyPayPalUser(masterSeller);
        PayPalUser buyer = UserBaseUtil.createUserWithFundingInstrumentsAndFlag(PPAccountType.PREMIER, country,
                currency, null, fundingInstrumentData);
        List<ValidationUnit> postPaymentValidations = validationUtility.build(Payment.CREATE,
                PaymentValidationTemplate.FUTURE_PAYMENTS_SALE, CreatePaymentTemplate.FUTUREPAYMENTS_SALE);

        filterCoffee.orchestrate(
                Payment.CREATE, CreatePaymentTemplate.FUTUREPAYMENTS_SALE, postPaymentValidations,
                Sale.GET,
                ClientChannel.FuturePaymentSale, buyer, seller);
    }
```

Looks like the first 2 lines are spent in standard user creation. The buyer, interestingly, has been parameterized with some fundingInstrumentData that is coming from a dataProvider. The author of this test wanted to ensure that this works for mutliple flavors of buyers of different countries and FI, so this is a succinct way to putting it together.

Next up there's a payment validation created. You see that there's a CreatePaymentTemplate that is passed along with a PaymentValidationTemplate as well. Presumably underneath, there's a validationBuilder that takes both of these and generates a proper list of validations. You dig deeper and find that the payload contains a list of transactinons which are parsed by the logic in the [PaymentValidationBuilderHelper](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/validation/builder/helper/PaymentValidationBuilderHelper.java#L500). In addition to this, the validationTemplate also provides cues to what additional fields to expect in the response.

You're curious about the auth related stuff that happens underneath. So you decide to dig deeper and see in the PaymentServiceImpl what gets orchestrated to call the Payment.CREATE api, and you [stumble upon this](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/dcc3bbca7242511c3b3bd2054a06dadcd1755a86/functional-tests/src/test/java/com/paypal/test/filtercoffee/service/impl/PaymentServiceImpl.java#L131). It's clear that devPlat has its own authCeremony which is managed independently. The next api call in the orchestration is a Sale.GET which seems pretty self-explanatory.


##Dynamic Payloads and Dynamic validations

### Dynamic Payloads
Gives you the ability to build a Payload based on inputs from previous API calls in the orchestraction
The Payload will be built before executing that ExecUnit

To create dynamic Payload you will have to extend the class [DynamicPayloadTemplate](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/template/payload/DynamicPayloadTemplate.java)
You can either create a concrete class or simply define a anonymous class 

Example

```java
   PayloadTemplate dynamicPayloadExample = new DynamicPayloadTemplate<>(new PayloadFromResponseMapper<Cart>() {

            @Override
            public Cart mapfromApiResponse(ConcurrentHashMap<Endpoint, Object> apiResponseMap) {
                Cart cart = new Cart();
                JsonNode addAddressResponse = (JsonNode) apiResponseMap
                        .get(UserPlatformServ.ADD_ADDRESS);
                cart.setAddressId(addAddressResponse.get("id").asText());
            }
        });
        
        
  //Add the payload to the orchestration
  call(CART.CREATE_CART,
                        dynamicPayloadExample);


```

Before execution on the ExecUnit, filtercoffee identifies the Dynamic Payload and invokes mapfromApiResponse method to get the custom payload which will be used as the request payload 

### Dynamic Validations

if you have a requirement to validate you API Response bases on previous responses, you can do so by using the [DynamicValidationUnitBuilder](https://github.paypal.com/Payments-R/paymentapiplatformserv/blob/develop/functional-tests/src/test/java/com/paypal/test/filtercoffee/source/DynamicValidationUnitBuilder.java)

The DynamicValidationUnitBuilder builds ValidationUnits on the fly based in the APIResponseMap passed in before the ExecUnit to be validated. 
You can always supply static validation units.The Dynamic ones are appended to the statically inputed validation units and both of them combined are run against the final response

```java

	//Create validation to verify the address on the checkout session is different
        DynamicValidationUnitBuilder checkoutSessionDynamicValidations = new DynamicValidationUnitBuilder() {
            @Override
            public List<ValidationUnit> buildValidationUnits(ExecutionUnit executionUnit,
                    ConcurrentHashMap<Endpoint, Object> apiResponseMap)
                    throws Exception {
                JsonNode addAddressResponse = (JsonNode) apiResponseMap
                        .get(UserPlatformServ.ADD_ADDRESS);

                List<ValidationUnit> validationUnits = new ArrayList<>();
                ValidationUnit vu = new ValidationUnit(CheckoutSessionResourceFields.ID.getField(),
                        addAddressResponse.get("id").asText(), CustomExpectation.NOT_EQUALS);
                validationUnits.add(vu);
                return validationUnits;
            }

        };
    
    //Add to the orchestration the dynamic Validation builder 
    .call(CheckoutSession.NOTIFY_SHIPPING,
                        NotifyShippingAddressTemplate.NOTIFY_DELETE_ADDRESS_TEMPLATE)
                .withDynamicValidationBuilder(checkoutSessionDynamicValidations)
    
    

```

### Custom Expectations in ValidationUnits

if case of a need to do custom assert against the expected field , you can always pass a custom expectation to the validation Unit you have built



```java

 public ValidationUnit(Field field, String expectedValue, CustomExpectation customExpectation) {
        super();
        this.field = field;
        this.expectedValue = expectedValue;
        this.customExpectation = customExpectation;
    }

```

Example Usage

```java
  
  *Predefined Custom expections*

  ValidationUnit vu = new ValidationUnit(CheckoutSessionResourceFields.ID.getField(),
                        addAddressResponse.get("id").asText(), CustomExpectation.NOT_EQUALS);
                        
                       
   *Custom Expectation*
   
     private final class AddressExistAndUpdatedInShippingAddressesExpectation extends CustomExpectation {
        @Override
        public void validate(Object responseObject, String expectedValue) {
            ShippingAddresses shippingAddresses = (ShippingAddresses) responseObject;
            boolean found = false;
            for (com.paypal.platform.payments.model.rest.common.ShippingAddress s : shippingAddresses) {
                if (StringUtils.equals(expectedValue, s.getId())) {
                    found = true;
                    Assert.assertTrue(s.isPreferredAddress(), "Address should be marked as prefered");
                }
            }
            Assert.assertTrue(found, "Recently added shipping address not found!");
        }
    }
    
     ValidationUnit vu = new ValidationUnit(ShippingAddressResourceFields.ID.getField(),
                    addAddressResponse.get("id").asText(), new AddressExistAndUpdatedInShippingAddressesExpectation());
                       

```

=======
## Tagging your tests

Now that you have created your tests and made sure they work, you need to make them fit in the bigger picture. Your test is only going to become relevant if it is part of a continuous integration suite. Fortunately this is easy to achieve as long as you follow some simple guidelines.

### 1. Put your test in the right folder ###

It is very important to make sure that your test is in the right folder in order for automation to run it. All tests should live under the parent folder com.paypal.test.filtercoffee.test and under the subfolder structure corresponding to the client and use case:

```java
package com.paypal.test.filtercoffee.test.[client].[usecase];
```
e.g. **com.paypal.test.filtercoffee.test.aries.guest** or **com.paypal.test.filtercoffee.test.prox.msp**

If in doubt, always check with your scrum master.
### 2. Tag your tests with the right groups ###
Now that your test lives in the right place it needs to be tagged such that automation can put it in the right bucket. To tag a test add a group to the test annotation:

```java
@Test(groups = { "P0Test", "RequiresConfigChange" })
    public void e2eShippingAddressTest() throws Exception {
        ...
        filterCoffee.orchestrate(
                ExpressCheckout.SET, SetECTemplate.SIMPLE_SALE_USD,
                CheckoutSession.CREATE,
                ...
                ExpressCheckout.DO, DoECTemplate.SIMPLE_SALE_USD, doECValidations,
                ClientChannel.AriesWebMember, buyer, seller);
    }
```

The following grouping is used in the symphony team:

- **P0Test** To be used used when this is the main test for a P0 use case. If you are not sure what a P0 means consult with your team.
- **P1Test** To be used used when this is the main test for a P1 use case. If you are not sure what a P1 means consult with your team.
- **RequiresConfigChange** To be used used when this test requires a config change in the stage as part of the test. For instance enabling a wowo, or making a cdb change.
- **RequiresDBChange** To be used used when this test requires a db update or insert as part of the test.
- **LongDuration** To be used used when this test uses a Thread.sleep or other mechanism to wait for some time. Or simply when the test takes abnormally long time to run always. 

Note that the groups are not mutually exclusive, a test can belong to one or more groups. Or none if none of the tags is applicable. You can also opt to add additional tags. But you are encouraged to communicate this to all the teams to determine if these new tags should become a standard.

### 3. Use the suite that suits you ###
If everybody has made the due diligence in tagging the tests appropriately creating a suite that fits your needs becomes very simple. For instance if you want to run all aries P0 test cases in a managed stage (that does not support DB/Config changes) you can specify the following in your testng xml suite:

```xml
<test verbose="2" name="SymphonyHermesTestSuite" annotations="JDK">
		<groups>
			<run>
				<include name="P0Test" />
				<exclude name="RequiresConfigChange" />
				<exclude name="RequiresDBChange" />
			</run>
		</groups>

		<packages>
			<package name="com.paypal.test.filtercoffee.test.aries.*" />
		</packages>
	</test>
```

Before you create a new suite file always check if there is not one created already.
