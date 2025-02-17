---
layout: post
title: "Write custom Assertions to improve your tests"
tags: tutorial csharp
cover: Cover.png
cover-alt: "Write custom Assertions to improve your tests"
---

Last time, we went through some best practices to [write better assertions]({% post_url 2021-07-19-WriteBetterAssertions %}) on our tests. This time, let's focus on how to use custom assertions to improve the readability of our tests.

**Use custom assertions to encapsulate multiple assertions on a single method and express them in the same language as the domain model. Write custom assertions with local methods or extension methods on the result object of the tested method or on the fake objects.**

We can either create custom assertions on top of the MSTest `Assert` class. And, our own `Verify` methods on Moq mocks. 

## 1. How to write custom MSTest Assert methods

Let's refactor one of our tests for [Stringie](https://github.com/canro91/Testing101), a (fictional) library to manipulate strings. We used Stringie to learn [4 common mistakes when writing tests]({% post_url 2021-03-29-UnitTestingCommonMistakes %}) and [4 test naming conventions]({% post_url 2021-04-12-UnitTestNamingConventions %}).

**To write custom assertions with MSTest, write an extension method on top of the Assert class. Then, compare the expected and actual parameters and throw an AssertFailedException if the comparison fails.**

Let's create a `StringIsEmpty()` method,

```csharp
internal static class CustomAssert
{
    public static void StringIsEmpty(this Assert assert, string actual)
    {
        if (string.IsNullOrEmpty(actual))
            return;

        throw new AssertFailedException($"Expect empty string but was {actual}");
    }
}
```

Then, we can use `StringIsEmpty()` with the `That` property. Like this,

```csharp
Assert.That.StringIsEmpty("");
```

With this custom assertion in place, we can rewrite the Assert part of our tests for the Stringie `Remove` method. Like this,

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;

namespace Stringie.UnitTests
{
    [TestClass]
    public class RemoveTests
    {
        [TestMethod]
        public void Remove_NoParameters_ReturnsEmpty()
        {
            string str = "Hello, world!";

            string transformed = str.Remove();

            Assert.That.StringIsEmpty(transformed);
            //          ^^^^^
        }
    }
}
```

With custom assertions, like `StringIsEmpty()`, we can write our assertions using the same vocabulary from our business domain.

<figure>
<img src="https://images.unsplash.com/photo-1414497729697-b8555ba6c1cc?crop=entropy&cs=tinysrgb&fit=crop&fm=jpg&h=400&ixid=MnwxfDB8MXxyYW5kb218MHx8fHx8fHx8MTYyNzA1MDM4Mw&ixlib=rb-1.2.1&q=80&utm_campaign=api-credit&utm_medium=referral&utm_source=unsplash_source&w=600" alt="Shaving wood" />

<figcaption>Photo by <a href="https://unsplash.com/@asthetik?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Mike Kenneally</a> on <a href="https://unsplash.com/s/photos/wood-workshop?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a></figcaption>
</figure>

## 2. How to write custom Moq Verify method

If we're using Moq, we can create our custom `Verify()` methods too.

Let's write some tests for an API client. This time, we have a payment processing system and we want to provide our users a user-friendly client to call our endpoints.

We want to test that our methods call the right API endpoints. If we change the client version number, we should include the version number in the endpoint URL.

We could write some unit tests like these ones,

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;
using Moq;
using System;
using System.Threading.Tasks;

namespace CustomAssertions
{
    [TestClass]
    public class PaymentProxyTests
    {
        [TestMethod]
        public async Task PayAsync_ByDefault_CallsLatestVersion()
        {
            var fakeClient = new Mock<IApiClient>();
            var proxy = new PaymentProxy(fakeClient.Object);

            await proxy.PayAsync(AnyPaymentRequest);

            // Here we verify we called the right url
            fakeClient.Verify(x => x.PostAsync<PaymentRequest, ApiResult>(
                It.Is<Uri>(t => t.AbsoluteUri.Contains("/v2/pay", StringComparison.InvariantCultureIgnoreCase)), It.IsAny<PaymentRequest>()),
                Times.Once);
        }

        [TestMethod]
        public async Task PayAsync_VersionNumber_CallsEndpointWithVersion()
        {
            var fakeClient = new Mock<IApiClient>();
            var proxy = new PaymentProxy(fakeClient.Object, Version.V1);

            await proxy.PayAsync(AnyPaymentRequest);

            // Here we verify we called the right url again
            fakeClient.Verify(x => x.PostAsync<PaymentRequest, ApiResult>(
                It.Is<Uri>(t => t.AbsoluteUri.Contains("/v1/pay", StringComparison.InvariantCultureIgnoreCase)), It.IsAny<PaymentRequest>()),
                Times.Once);
        }

        private PaymentRequest AnyPaymentRequest
            => new PaymentRequest
            {
                // All initializations here...
            };
    }
}
```

These tests rely on [Moq to write fakes]({% post_url 2020-08-11-HowToCreateFakesWithMoq %}) and [object mothers to create test data]({% post_url 2021-04-26-CreateTestValuesWithBuilders %}).

Notice, the `Verify()` methods in the two tests. Did you notice how buried inside all that boilerplate is the URL we want to check? That's what we're interested in. It would be nice if we had a `VerifyItCalled()` method and we just passed a string or URI with the URL we want.

Let's create an extension method on top of our fake. Let's write the `VerifyItCalled()` method we want. It will receive a relative URL and call Moq `Verify()` method. Something like this,

```csharp
using Moq;
using System;

namespace CustomAssertions
{
    public static class MockApiClientExtensions
    {
        public static void VerifyItCalled(this Mock<IApiClient> mock, string relativeUri)
        {
            mock.Verify(x => x.PostAsync<PaymentRequest, ApiResult>(
                            It.Is<Uri>(t => t.AbsoluteUri.Contains(relativeUri, StringComparison.InvariantCultureIgnoreCase)),
                            It.IsAny<PaymentRequest>()),
                        Times.Once);
        }
    }
}
```

With our `VerifyItCalled()` in place, let's refactor our tests to use it,

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;
using Moq;
using System.Threading.Tasks;

namespace CustomAssertions
{
    [TestClass]
    public class PaymentProxyTests
    {
        [TestMethod]
        public async Task PayAsync_ByDefault_CallsLatestVersion()
        {
            var fakeClient = new Mock<IApiClient>();
            var proxy = new PaymentProxy(fakeClient.Object);

            await proxy.PayAsync(AnyPaymentRequest);

            // Now, it's way more readable
            fakeClient.VerifyItCalled("/v2/pay");
            //         ^^^^^
        }

        [TestMethod]
        public async Task PayAsync_VersionNumber_CallsEndpointWithVersion()
        {
            var fakeClient = new Mock<IApiClient>();
            var proxy = new PaymentProxy(fakeClient.Object, Version.V1);

            await proxy.PayAsync(AnyPaymentRequest);

            // No more boilerplate code to check things
            fakeClient.VerifyItCalled("/v1/pay");
            //         ^^^^^
        }

        private PaymentRequest AnyPaymentRequest
            => new PaymentRequest
            {
                // All initializations here
            };
    }
}
```

With our custom `Verify()` method, our tests are more readable. And, we wrote the tests in the same terms as our domain language. No more ceremony to check we called the right url.

Voilà! That's how to use custom assertions to write tests in the same terms as your domain model.

In case you have plain assertions, not `Verify()` methods with Moq, simply write private methods to group your assertions and share them in a base test class. Or write extension methods on the output of the method being tested. For more details on this technique, check xUnitPatterns on [Custom Assertions](http://xunitpatterns.com/Custom%20Assertion.html).

If you're new to fakes, mocks and stubs, read [What are fakes in unit testing]({% post_url 2021-05-24-WhatAreFakesInTesting %}) and [how to write better stubs and mocks]({% post_url 2021-06-07-TipsForBetterStubsAndMocks %}). Also, don't miss my [Unit testing best practices]({% post_url 2021-07-05-UnitTestingBestPractices %}) and [how to write better assertions]({% post_url 2021-07-19-WriteBetterAssertions %}).

_Happy testing!_