---
layout: post
title: "How to write good unit tests: Noise and Hidden Test Values"
tags: tutorial csharp
cover: WriteUnitTests.png
cover-alt: Two issues to avoid to write good unit tests
---

These days, I needed to update some unit tests. I found two types of issues with them. Please, continue to read. Maybe, you're a victim of those issues, too. Let's learn how to write good unit tests.

**To write good unit tests, avoid complex setup scenarios and hidden test values. Often tests are bloated with unneeded or complex code in the Arrange part and full of magic or hidden test values. Unit tests should be even more readable than production code**.

## The tests I had to update

The tests I needed to update were for an ASP.NET Core API controller, `AccountController`. This controller created, updated, and suspended user accounts. Also, it sent a welcome email to new users.

These tests checked a configuration object for the sender, reply-to, and contact-us email addresses. The welcome email contained those three emails. If the configuration files miss one of the email addresses, the controller throws an exception from its constructor.

Let's see one of the tests. This test checks for the sender's email.

```csharp
[TestMethod]
public Task AccountController_SenderEmailIsNull_ThrowsException()
{
    var mapper = new Mock<IMapper>();
    var logger = new Mock<ILogger<AccountController>>();
    var accountService = new Mock<IAccountService>();
    var accountPersonService = new Mock<IAccountPersonService>();
    var emailService = new Mock<IEmailService>();
    var emailConfig = new Mock<IOptions<EmailConfiguration>>();
    var httpContextAccessor = new Mock<IHttpContextAccessor>();

    emailConfig.SetupGet(options => options.Value)
        .Returns(new EmailConfiguration()
        {
            ReplyToEmail = "email@email.com",
            SupportEmail = "email@email.com"
        });

    Assert.ThrowsException<ArgumentNullException>(() =>
        new AccountController(
            mapper.Object,
            logger.Object,
            accountService.Object,
            accountPersonService.Object,
            emailService.Object,
            emailConfig.Object,
            httpContextAccessor.Object
        ));

    return Task.CompletedTask;
}
```

This test uses [Moq to create stubs and mocks]({% post_url 2020-08-11-HowToCreateFakesWithMoq %}).

Can you spot any issues in our sample test? The naming convention isn't one, by the way.

Let's see the two issues to avoid to write good unit tests.

<figure>
<img src="https://images.unsplash.com/32/6Icr9fARMmTjTHqTzK8z_DSC_0123.jpg?ixlib=rb-1.2.1&q=80&fm=jpg&crop=entropy&cs=tinysrgb&w=800&h=400&fit=crop&ixid=eyJhcHBfaWQiOjF9" alt="Adjusting dials on a mixer" />

<figcaption><span>Photo by <a href="https://unsplash.com/@drewpatrickmiller?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Drew Patrick Miller</a> on <a href="https://unsplash.com/photos/73o_FzZ5x-w?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span></figcaption>
</figure>

## 1. Reduce the noise

Our sample test only cares about one object: `IOptions<EmailConfiguration>`. All other objects are noise for our test. They don't have anything to do with the scenario under test. We have to use them to make our test compile.

**Use builder methods to reduce complex setup scenarios.**

Let's reduce the noise from our test with a `MakeAccountController()` method. It will receive the only parameter the test needs.

After this change, our test looked like this:

```csharp
[TestMethod]
public void AccountController_SenderEmailIsNull_ThrowsException()
//     ^^^^
// We can make this test a void method
{
    var emailConfig = new Mock<IOptions<EmailConfiguration>>();
    emailConfig
        .SetupGet(options => options.Value)
        .Returns(new EmailConfiguration
        {
            ReplyToEmail = "email@email.com",
            SupportEmail = "email@email.com"
        });

    // Notice how we reduced the noise with a builder
    Assert.ThrowsException<ArgumentNullException>(() =>
        MakeAccountController(emailConfig.Object));
        // ^^^^^
    
    // We don't need a return statement here anymore
}

private AccountController MakeAccountController(IOptions<EmailConfiguration> emailConfiguration)
{
    var mapper = new Mock<IMapper>();
    var logger = new Mock<ILogger<AccountController>>();
    var accountService = new Mock<IAccountService>();
    var accountPersonService = new Mock<IAccountPersonService>();
    var emailService = new Mock<IEmailService>();
    // We don't need Mock<IOptions<EmailConfiguration>> here
    var httpContextAccessor = new Mock<IHttpContextAccessor>();

    return new AccountController(
            mapper.Object,
            logger.Object,
            accountService.Object,
            accountPersonService.Object,
            emailService.Object,
            emailConfiguration,
            // ^^^^^
            httpContextAccessor.Object);
}
```

Also, since our test doesn't have any asynchronous code, we could declare our test as a `void` method and remove the `return` statement. That looked weird in a unit test, in the first place.

With this refactor, our test started to look simpler and easier to read. Now, it's clear this test only cares about the `EmailConfiguration` class.

## 2. Make your test values obvious

Our test states in its name that the sender's email is null. Anyone reading this test would expect to see a variable set to null and passed around. But, that's not the case.

**Make scenarios under test and test values extremely obvious**.

Please, don't make developers decode your tests.

To make the test scenario obvious in our example, let's add `SenderEmail = null` to the initialization of the `EmailConfiguration` object.

```csharp
[TestMethod]
public void AccountController_SenderEmailIsNull_ThrowsException()
{
    var emailConfig = new Mock<IOptions<EmailConfiguration>>();
    emailConfig
        .SetupGet(options => options.Value)
        .Returns(new EmailConfiguration
        {
            // The test value is obvious now
            SenderEmail = null,
            //            ^^^^^
            ReplyToEmail = "email@email.com",
            SupportEmail = "email@email.com"
        });

    Assert.ThrowsException<ArgumentNullException>(() =>
        MakeAccountController(emailConfig.Object));
}
```

If we have similar scenarios, we can use a constant like `const string NoEmail = null`. Or prefer [object mothers and builders to create test data]({% post_url 2021-04-26-CreateTestValuesWithBuilders %}).

### Don't write mocks for IOptions<T>

Finally, as an aside, we don't need a mock on `IOptions<EmailConfiguration>`.

**Don't use a mock or a stub with the IOptions interface. That would introduce extra complexity. Use Options.Create() with the value to configure instead.**

Let's use the `Option.Create()` method instead.

```csharp
[TestMethod]
public void AccountController_SenderEmailIsNull_ThrowsException()
{
    var emailConfig = Options.Create(new EmailConfiguration
    //                ^^^^^
    {
        SenderEmail = null,
        ReplyToEmail = "email@email.com",
        SupportEmail = "email@email.com"
    });

    Assert.ThrowsException<ArgumentNullException>(() =>
        MakeAccountController(emailConfig));
}
```

Voilà! That's way easier to read. Do you have noise and hidden test values in your tests? Remember, readability is one of the pillars of unit testing. Don't make developers decode your tests.

For other tips on writing good unit tests, check my follow-ups on [writing failing tests first]({% post_url 2021-02-05-FailingTest %}) and [using simple test values]({% post_url 2022-12-14-SimpleTestValues %}). Also, don't miss my [Unit Testing 101 series]({% post_url 2021-08-30-UnitTesting %}) where I cover more subjects like this.

_Happy unit testing!_