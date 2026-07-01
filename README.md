# Selenium Framework using Spring Boot

A robust test automation framework built on **Java Spring Boot**, integrating **Selenium WebDriver** and **TestNG** to create scalable, maintainable web-based automation scripts.

---

## Key Features

- **Parallel Test Execution** — Thread-safe test runs with efficient thread management
- **Random Test Data Generation** — Powered by the Java Faker API
- **Spring Dependency Injection** — Clean, decoupled object creation
- **WebDriverManager Integration** — Automatic browser driver management
- **Platform Independent** — Runs consistently across environments
- **Allure Reports** — Rich, interactive test reporting
- **SonarQube Integration** — Static code analysis and vulnerability scanning
- **Lombok Support** — Reduces boilerplate code

---

## Table of Contents

- [Installation](#installation)
- [Test Framework Design Details](#test-framework-design-details)
- [Test Execution Flow](#test-execution-flow)
- [Folder Structure](#folder-structure)
- [Running the Tests](#running-the-tests)
- [Creating New Tests](#creating-new-tests)
- [Generating Sonar Reports](#generating-sonar-reports)
- [Built With](#built-with)
- [Contributing](#contributing)
- [License](#license)
- [References](#references)

---

## Test Framework Design Details

> A visual flowchart of the framework architecture is available in the `/docs` folder of this repository (see `flowchart.png`).

The framework follows a layered design:

- **Test Layer** — TestNG test classes annotated with Spring Boot test context
- **Page Object Layer** — Encapsulated page interactions using the Page Object / Page Fragment pattern
- **Action Layer** — Shared, reusable Selenium actions via interfaces (`IUIElements`, `IElementVerification`)
- **Configuration Layer** — Spring-managed WebDriver lifecycle, thread scoping, and application properties
- **Reporting Layer** — Allure integration for step-level reporting and screenshots on failure

---

## Test Execution Flow

1. Execution starts from `pom.xml`.
2. Control passes to the parameterized `suites.xml`.
3. TestNG invokes the methods in each suite in **parallel mode**, per the XML configuration (a separate thread is created for each parallel run).
4. Control moves to the `ElementTests` test class.
5. Spring's dependency injection creates the `BasePage` and `ElementsPage` objects.
6. The `@BeforeMethod` annotation prepares the Spring test context.
7. Execution begins with the `sanityCheck()` method, based on group dependencies defined in the `@Test` annotations.
8. Control transfers to the TestNG listener registered via `@Listeners(TestListener.class)` at the class level.
9. The overridden `onTestStart` method logs the start of test execution.
10. Control returns to the test to execute `openElementsPage()`.
11. Control passes to the `ElementsPage` class.
12. The `openURL` method is invoked via the `iUIElements` object (implementation of the `IUIElements` interface).
13. The WebDriver instance is created **only when required**, since `IUIElements` implementations inherit from `ActionsBaseClass`, which is tagged with `@Lazy`.
14. Driver creation is handled by the Spring `@Configuration` class `WebdriverConfig`.
15. Using `@ConditionalOnExpression`, a WebDriver bean is created based on the `browser` property (e.g., `chrome`, `edge`) defined in `application.properties`.
16. Once created, the bean's lifecycle is bound to the TestNG thread via the `@Scope("driverscope")` annotation.
17. The custom `driverscope` is registered with Spring by extending `BeanFactoryPostProcessor`.
18. This is implemented through the `DriverScopeConfig` and `DriverScope` classes, where `DriverScope` extends `SimpleThreadScope`.
19. The same `@Scope("driverscope")` annotation is used to create a `WebDriverWait` bean, shared across action classes.
20. Once the bean is instantiated, a browser session starts and navigates to the URL defined in the constants file.
21. The page title is extracted using the `iElementVerification` object (implementation of `IElementVerification`).
22. The retrieved value is asserted against the expected value to determine pass/fail status.
23. Based on the assertion result, control transfers to `TestListener` to mark the test as passed or failed.
24. Results are pushed to the Allure report via the `testReportUpdate` method.
25. The `@AfterMethod(alwaysRun = true)` annotation triggers `teardown()` in `BasePage` to close the browser.
26. Screenshots are captured and embedded into the Allure report.
27. After the sanity check completes, `textBoxVal_TC001()` executes.
28. Test data is injected via the `@DataProvider` annotation, generated using the Java Faker API in `UserDataProvider`.
29. Page actions are performed through the `TextBoxPF` page fragment class.
30. In parallel, `checkBoxVal__TC002()` executes (per TestNG configuration) via the `CheckBoxPF` page fragment class.

> System and application properties can be updated directly in `application.properties`, or overridden via Maven command-line arguments.

---

## Folder Structure

> A detailed folder structure diagram is available in the `/docs` folder of this repository.

```
src
├── main
│   └── java/com/auto/framework
│       ├── config          # Spring configuration (WebDriver, DriverScope)
│       ├── constants        # Application constants (URLs, etc.)
│       ├── listeners         # TestNG listeners
│       ├── pageobjects       # Page Objects and Page Fragments
│       └── testdata          # Data providers and models
└── test
    └── resources
        ├── application.properties
        └── suites.xml
```

---

## Installation

Clone the repository:

```bash
git clone https://github.com/naveenchr/SpringBoot-SeleniumFramework.git
```

Clean and compile the Maven project:

```bash
mvn clean
mvn compile
```

---

## Running the Tests

From the command line:

```bash
mvn test
```

Parameters can be overridden at runtime instead of editing `application.properties` directly:

```bash
mvn test -Dmy.properties.grid-url=https://test.com/wd/hub -Dmy.properties.grid-token=123456 -Dmy.properties.grid=true
```

### Viewing the Report

1. Navigate to the `target` folder.
2. Run the following command:

```bash
allure serve -h localhost
```

> Sample report screenshots are available in the `/docs` folder of this repository.

---

## Creating New Tests

### Test Class Example

```java
package com.auto.framework;

import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.MatcherAssert.assertThat;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.testng.AbstractTestNGSpringContextTests;
import org.testng.annotations.AfterMethod;
import org.testng.annotations.BeforeMethod;
import org.testng.annotations.Listeners;
import org.testng.annotations.Test;
import com.auto.framework.listeners.TestListener;
import com.auto.framework.pageobjects.common.BasePage;
import com.auto.framework.pageobjects.demoqa.ElementsPage;
import com.auto.framework.testdata.UserDataProvider;
import com.auto.framework.testdata.UserModal;

@SpringBootTest
@Listeners(TestListener.class)
public class ElementTests extends AbstractTestNGSpringContextTests {

    @Autowired
    private BasePage basePage;

    @Autowired
    public ElementsPage elementsPage;

    @Test(groups = "Sanity Test")
    public void sanityCheck() {
        elementsPage.openElementsPage();
        assertThat(elementsPage.getPageTitle(), is("DEMOQA"));
    }

    @Test(dependsOnGroups = "Sanity Test", dataProvider = "User Data", dataProviderClass = UserDataProvider.class)
    public void textBoxVal__TC001(UserModal userData) {
        // Opens browser page
        elementsPage.textBoxPF.openTextBoxPage();

        // Perform testing actions
        elementsPage.textBoxPF.enterFullname(userData.getFirstName());
        elementsPage.textBoxPF.enterEmail(userData.getEmail());
        elementsPage.textBoxPF.enterCurrentAddress(userData.getCurrAddress());
        elementsPage.textBoxPF.enterPermanentAddress(userData.getPermAddress());
        elementsPage.textBoxPF.submitForm();

        // Assert data points
        assertThat(elementsPage.textBoxPF.getConfirmationMessage().getFirstName(), is(userData.getFirstName()));
        assertThat(elementsPage.textBoxPF.getConfirmationMessage().getEmail(), is(userData.getEmail()));
        assertThat(elementsPage.textBoxPF.getConfirmationMessage().getCurrAddress(), is(userData.getCurrAddress()));
        assertThat(elementsPage.textBoxPF.getConfirmationMessage().getPermAddress(), is(userData.getPermAddress()));
    }

    @BeforeMethod
    @Override
    public void springTestContextPrepareTestInstance() throws Exception {
        super.springTestContextPrepareTestInstance();
    }

    @AfterMethod(alwaysRun = true)
    public void teardownDriver() {
        basePage.teardownDriver();
    }
}
```

### Page Object Example

```java
package com.auto.framework.pageobjects.demoqa;

import static com.auto.framework.constants.Constants.ELEMENTS_PAGE;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import com.auto.framework.pageobjects.common.BasePage;
import io.qameta.allure.Step;

@Component
public class ElementsPage extends BasePage {

    @Autowired
    public TextBoxPF textBoxPF;

    @Autowired
    public CheckBoxPF checkBoxPF;

    @Autowired
    public RadioButtonPF radioButtonPF;

    @Autowired
    public WebTablePF webTablePF;

    @Step("Open webpage")
    public void openElementsPage() {
        iUIElements.openURL(myProperties.getDemoUrl() + ELEMENTS_PAGE);
    }

    @Step("Verify Page Title")
    public String getPageTitle() {
        return iElementVerification.getTitle();
    }
}
```

---

## Generating Sonar Reports

1. Add the SonarQube server URL to your `pom.xml`:

```xml
<sonar.host.url>http://localhost:9000</sonar.host.url>
```

2. Run the following Maven command after test execution:

```bash
mvn sonar:sonar
```

---

## Built With

- [Spring Boot](https://spring.io/projects/spring-boot)
- [Selenium WebDriver](https://www.selenium.dev/)
- [TestNG](https://testng.org/)
- [Maven](https://maven.apache.org/)
- [Allure Reports](https://qameta.io/allure-report/)
- [Hamcrest](http://hamcrest.org/)
- [Lombok](https://projectlombok.org/)
- [Java Faker](https://github.com/DiUS/java-faker)

---

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

---

## License

This project is licensed under the terms specified in the `LICENSE` file included in this repository.

---

## References

- [Selenium Documentation](https://www.selenium.dev/documentation/)
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/documentation.html)
- [TestNG Documentation](https://testng.org/doc/)
- [Allure Framework Documentation](https://docs.qameta.io/allure/)