# Contribution guidelines
 
## Recognized use-cases

This test harness is maintained with following use-cases in mind. In case you are interested in other usage, there is no guarantee it will work as expected or that the particular use-case will not be broken by future updates. Please stick with recognized use-cases or propose your use-case to be declared here to be on the safe side.

- Jenkins core PR verification testsuite.
- Library for UI testing used by Jenkins components.
- Periodic verification of the health of the upstream Jenkins distribution.
  - Latest releases of components are being tested together to spot problems early.
- Verification of release candidates and their compatibility with plugins.
- Verification of custom combination of components (Jenkins core and plugin versions).
  - Pre-deployment tests.
  - Internal update center verification.

## Test contribution

In order for tests to be added they should:

- have significant user-base (prominent core feature, feature of a plugin installed by at least 1% of deployments)
- represent realistic, nontrivial and fundamental use-case
- covering sufficiently distinct use-case than other ATH tests
- plugin tests should have whole testsuite executed in less than 20 minutes on public CI server

Existing tests that do not adhere to these guidelines will be reevaluated and proposed for restructure or move to the component test suite by test/component maintainers. In case of not enough interest, the test will be dropped.

## Dealing with fragile/broken tests

Test is determined fragile/broken when it is failing despite no defects are present in tested component.

- JIRA issue will be created notifying author of the test or plugin authors.
  - Test can be fixed or moved away to the [sources of the plugin](EXTERNAL.md).
- Test would be `@Ignore`d while the JIRA is open
- At the end of the time period, the test would be removed from ATH.
  - Test can still be resurrected from history if needed - in ATH or plugin itself.

The situation is expected to be handled in 90 days since reported. In case of no corrective action, the test will be removed from ATH.

## Guidelines to avoid fragile/flaky tests

### Click and wait

When writing acceptance tests using Selenium, it is common to click on an element that modifies the current page and then run some other action on the resulting page.
Most of the time the page is updated really quick (specially if there is not a network request involved), but on slow infrastructure this can take some time so that the next test action after clicking could be checking on something which is not the expected.

To avoid this, use the click and wait pattern:

```
WebElement clickableElement = ...;
clickableElement.click();
// "my-element" is the element you expect visible in the DOM after click
waitFor(by.id("my-element"), 10); // wait with 10 sec timeout
// now run actions safely on my-element
```

In the test API there are some DOM search methods that already use this pattern, like `CapybaraPortingLayerImpl#find(By selector)`.
These find methods are also checking for visible elements, as opposed to plain `waitFor` which is just checking if the element is in the DOM (visible ot not).

This way the test will not be affected by an underlying slow infrastructure.

### Do not rely on external infra

When the test requires to interact with an external system, use a `DockerFixture`.
There are multiple examples into the [`fixtures` directory](https://github.com/jenkinsci/acceptance-test-harness/tree/master/src/main/java/org/jenkinsci/test/acceptance/docker/fixtures).

### Do not use sleep

Sleeping blindly is a bad practice, it just makes tests execution time longer. 
Use the [`Wait` class](https://github.com/jenkinsci/acceptance-test-harness/blob/master/src/main/java/org/jenkinsci/test/acceptance/junit/Wait.java) as much as possible. `CapybaraPortingLayerImpl#waitFor` is an easy way to create instances of `Wait`.

### Use robust selectors

When selecting elements from the DOM, use robust selectors. 
In general, try to avoid selectors that are known to change or are outside your control (they are not in your code). 

A few examples:

1. Do not use full xpath expressions. DOM can change anytime.
2. Avoid using parts of the DOM which are not under your control (a change in core or any other plugin can make your test to fail).
3. Use paths generated by `form-element-path` plugin as much as possible (installed by default by the harness code). Look for the `path` attribute in DOM elements.

Another option to get robust selectors is the use of `id` on specific elements. If you really need to use them (because any other selector is not good enough) keep in mind that Jenkins is a pluggable system so there could be other plugins injecting ids also. Make sure you use unique ids (i.e. prepending your plugin ID to the HTML id).

### Use existing PageObjects

If your test needs to interact with Jenkins Core (or other plugins UI), check if there is already a PageObject for it instead of writing the code by yourself.
Existing PageObjects are usually well tested so they are possibly more stable than your newly added code.

Also, try to create PageObjects to interact with the UI instead of hardcoding elements searches and interactions in your test.
These PageObjects can be reused by others and they are much more maintainable as it's clear what's the UI they are handling.
