# Spring Boot Errors [![Build Status](https://travis-ci.org/alimate/errors-spring-boot-starter.svg?branch=master)](https://travis-ci.org/alimate/errors-spring-boot-starter) [![codecov](https://codecov.io/gh/alimate/errors-spring-boot-starter/branch/master/graph/badge.svg)](https://codecov.io/gh/alimate/errors-spring-boot-starter) [![Maven Central](https://img.shields.io/maven-central/v/me.alidg/errors-spring-boot-starter.svg)](https://search.maven.org/search?q=g:me.alidg%20AND%20a:errors-spring-boot-starter)
A Bootiful, consistent and opinionated approach to handle all sorts of exceptions.

## Table of Contents

  * [Make error handling great again!](#make-error-handling-great-again)
  * [Getting Started](#getting-started)
    + [Download](#download)
    + [Prerequisites](#prerequisites)
    + [Overview](#overview)
    + [Error Codes](#error-codes)
    + [Error Message](#error-message)
    + [Exposing Arguments](#exposing-arguments)
    + [Validation and Binding Errors](#validation-and-binding-errors)
    + [Custom Exceptions](#custom-exceptions)
    + [Spring MVC](#spring-mvc)
    + [Spring Security](#spring-security)
    + [Customizing the Error Representation](#customizing-the-error-representation)
    + [Default Error Handler](#default-error-handler)
    + [Refining Exceptions](#refining-exceptions)
    + [Registering Custom Handlers](#registering-custom-handlers)
    + [Enabling Web MVC Test Support](#enabling-web-mvc-test-support)
  * [License](#license)

## Make error handling great again!
Built on top of Spring Boot's great exception handling mechanism, the `errors-spring-boot-starter` offers:
 - A consistent approach to handle all exceptions. Doesn't matter if it's a validation/binding error or a 
 custom domain-specific error or even a Spring related error, All of them would be handled by a `WebErrorHandler`
 implementation (No more `ErrorController` vs `@ExceptionHandler`)
 - Built-in support for application specific error codes, again, for all possible errors.
 - Simple error message interpolation using `MessageSource`.
 - Customizable HTTP error representation.
 - Exposing arguments from exceptions to error messages.

## Getting Started

### Download

Download the [latest JAR](https://search.maven.org/remotecontent?filepath=me/alidg/errors-spring-boot-starter/1.1.0/errors-spring-boot-starter-1.1.0.jar) or grab via Maven:

```
<dependency>
    <groupId>me.alidg</groupId>
    <artifactId>errors-spring-boot-starter</artifactId>
    <version>1.1.0</version>
</dependency>
```

or Gradle:
```
compile "me.alidg:errors-spring-boot-starter:1.1.0"
```

### Prerequisites
The main dependency is JDK 8+. Tested with:
 - JDK 8, JDK 9 and JDK 10 on both Linux and OSX
 - Spring Boot `2.0.5.RELEASE`

### Overview
The `WebErrorHandler` implementations are responsible for handling different kinds of exceptions. When an exception 
happens, the `WebErrorHandlers` (A factory over all `WebErrorHandler` implementations) catches the exception and would 
find an appropriate implementation to handle the exception. By default, `WebErrorHandlers` consults with the
following implementations to handle a particular exception:
 - An implementation to handle all validation/binding exceptions.
 - An implementation to handle custom exceptions annotated with the `@ExceptionMapping`.
 - An implementation to handle Spring MVC specific exceptions.
 - And if the Spring Security is on the classpath, An implementation to handle Spring Security specific exceptions.

After delegating to the appropriate handler, the `WebErrorHandlers` turns the handled exception result into a `HttpError`,
which encapsulates the HTTP status code and all error code/message combinations.

### Error Codes
Although using appropriate HTTP status codes is a recommended approach in RESTful APIs, sometimes, we need more information 
to find out what exactly went wrong. This is where *Error Codes* comes in. You can think of an error code as a *Machine Readable* 
description of the error. Each exception can be mapped to **at least one** error code.

In `errors-spring-boot-starter`, one can map exceptions to error codes in different ways:
 - Validation error codes can be extracted from the *Bean Validation*'s constraints:
     ```java
     public class User {  
    
         @NotBlank(message = "username.required")
         private final String username;
      
         @NotBlank(message = "password.required")
         @Size(min = 6, message = "password.min_length")
         private final String password;
      
         // constructor and getter and setters
     }
     ```
    To report a violation in password length, the `password.min_length` would be reported as the error code. As you may guess,
    one validation exception can contain multiple error codes to report all validation violations at once.
    
 - Specifying the error code for custom exceptions using the `@ExceptionMapping` annotation:
    ```java
    @ExceptionMapping(statusCode = BAD_REQUEST, errorCode = "user.already_exists")
    public class UserAlreadyExistsException extends RuntimeException {}
    ```
    The `UserAlreadyExistsException` exception would be mapped to `user.already_exists` error code.
 
 - Specifying the error code in a `WebErrorHandler` implementation:
    ```java
    public class ExistedUserHandler implements WebErrorHandler {
    
        @Override
        public boolean canHandle(Throwable exception) {
            return exception instanceof UserAlreadyExistsException;
        }
     
        @Override
        public HandledException handle(Throwable exception) {   
            return new HandledException("user.already_exists", BAD_REQUEST, null);
        }
    }
    ```

### Error Message
Once the exception mapped to error code(s), we can add a companion and *Human Readable* error message. This can be done
by registering a Spring `MessageSource` to perform the *code-to-message* translation. For example, if we add the following
key-value pair in our message resource file:
```properties
user.already_exists=Another user with the same username already exists
```
Then if an exception of type `UserAlreadyExistsException` was thrown, you would see a `400 Bad Request` HTTP response 
with a body like:
```json
{
  "errors": [
    {
      "code": "user.already_exists",
      "message": "Another user with the same username already exists"
    }
  ]
}
```
Since `MessageSource` supports Internationalization (i18n), our error messages can possibly have different values based
on each *Locale*.

### Exposing Arguments
With *Bean Validation* you can pass parameters from the constraint validation, e.g. `@Size`, to its corresponding 
interpolated message. For example, if we have:
```properties
password.min_length=The password must be at least {0} characters
```
And a configuration like:
```java
@Size(min = 6, message = "password.min_length")
private final String password;
```
The `min` attribute from the `@Size` constraint would be passed to the message interpolation mechanism, so:
```json
{
  "errors": [
    {
      "code": "password.min_length",
      "message": "The password must be at least 6 characters"
    }
  ]
}
```
In addition to support this feature for validation errors, we extend it for custom exceptions using the `@ExposeAsArg`
annotation. For example, if we're going to specify the already taken username in the message:
```properties
user.already_exists=Another user with the '{0}' username already exists
```
We could write:
```java
@ExceptionMapping(statusCode = BAD_REQUEST, errorCode = "user.already_exists")
public class UserAlreadyExistsException extends RuntimeException {
    @ExposeAsArg(0) private final String username;
    
    // constructor
}
```
Then the `username` property from the `UserAlreadyExistsException` would be available to the message under the 
`user.already_exists` key as the first argument. `@ExposeAsArg` can be used on fields and no-arg methods with a
return type.
The `HandledException` class also accepts the *to-be-exposed* arguments in its constructor.

### Validation and Binding Errors
Validation errors can be processed as you might expect. For example, if a client passed an empty JSON to a controller method
like:
```java
@PostMapping
public void createUser(@RequestBody @Valid User user) {
    // omitted
}
```
Then the following error would be returned:
```json
{
  "errors": [
    {
      "code": "password.min_length",
      "message": "corresponding message!"
    },
    {
       "code": "password.required",
       "message": "corresponding message!"
    },
    {
      "code": "username.required",
      "message": "corresponding message!"
    }
  ]
}
```

### Custom Exceptions
Custom exceptions can be mapped to status code and error code combination using the `@ExceptionMapping` annotation:
```java
@ExceptionMapping(statusCode = BAD_REQUEST, errorCode = "user.already_exists")
public class UserAlreadyExistsException extends RuntimeException {}
```
Here, every time we catch an instance of `UserAlreadyExistsException`, a `Bad Request` HTTP response with `user.already_exists`
error would be returned.

Also, it's possible to expose some arguments from custom exceptions to error messages using the `ExposeAsArg`:
```java
@ExceptionMapping(statusCode = BAD_REQUEST, errorCode = "user.already_exists")
public class UserAlreadyExistsException extends RuntimeException {
    @ExposeAsArg(0) private final String username;
    
    // constructor
    
    @ExposeAsArg(1)
    public String exposeThisToo() {
        return "42";
    }
}
```
Then the error message template can be something like:
```properties
user.already_exists=Another user exists with the '{0}' username: {1}
```
During message interpolation, the `{0}` and `{1}` placeholders would be replaced with annotated field's value and
method's return value. The `ExposeAsArg` annotation is applicable to:
 - Fields
 - No-arg methods with a return type

### Spring MVC
By default, a custom `WebErrorHandler` is registered to handle common exceptions thrown by Spring MVC:

|                 Exception                 | Status Code |            Error Code         |               Exposed Args               |
|:-----------------------------------------:|:-----------:|:-----------------------------:|:----------------------------------------:|
| `HttpMessageNotReadableException`         |     400     | `web.invalid_or_missing_body` | -                                        |
| `HttpMediaTypeNotAcceptableException`     |     406     | `web.not_acceptable`          | List of acceptable MIME types            |
| `HttpMediaTypeNotSupportedException`      |     415     | `web.unsupported_media_type`  | The unsupported content type             |
| `HttpRequestMethodNotSupportedException`  |     405     | `web.method_not_allowed`      | The invalid HTTP method                  |
| `MissingServletRequestParameterException` |     400     | `web.missing_parameter`       | Name and type of the missing query Param |
| `MissingServletRequestPartException`      |     400     | `web.missing_part`            | Missing request part name                |
| `NoHandlerFoundException`                 |     404     | `web.no_handler`              | The request path                         |
| `others`                                  |     500     | `unknown_error`               | -                                        |

### Spring Security
When Spring Security is present on the classpath, a `WebErrorHandler` implementation would be responsible to handle
common Spring Security exceptions:

|                   Exception                  | Status Code |         Error Code         |
|:--------------------------------------------:|:-----------:|:--------------------------:|
| `AccessDeniedException`                      |     403     | `security.access_denied`   |
| `AccountExpiredException`                    |     400     | `security.account_expired` |
| `AuthenticationCredentialsNotFoundException` |     401     | `security.auth_required`   |
| `AuthenticationServiceException`             |     500     | `security.internal_error`  |
| `BadCredentialsException`                    |     400     | `security.bad_credentials` |
| `UsernameNotFoundException`                  |     400     | `security.bad_credentials` |
| `InsufficientAuthenticationException`        |     401     | `security.auth_required`   |
| `LockedException`                            |     400     | `security.user_locked`     |
| `DisabledException`                          |     400     | `security.user_disabled`   |
| `others`                                     |     500     | `unknown_error`            |

### Customizing the Error Representation
By default, errors would manifest themselves in the HTTP response bodies with the following JSON schema:
```json
{
  "errors": [
    {
      "code": "the_error_code",
      "message": "the_error_message"
    }
  ]
}
```
In order to change this representation, just implement the `HttpErrorAttributesAdapter` interface and register it as
*Spring Bean*:
```java
@Component
public class OopsDrivenHttpErrorAttributesAdapter implements HttpErrorAttributesAdapter {
    
    @Override
    public Map<String, Object> adapt(HttpError httpError) {
        return Collections.singletonMap("Oops!", httpError);
    }
}
```

### Default Error Handler
By default, when all registered `WebErrorHandler`s refuse to handle a particular exception, the `LastResortWebErrorHandler`
would catch the exception and return a `500 Internal Server Error` with `unknown_error` as the error code.

If you don't like this behavior, you can change it by registering a *Bean* of type `WebErrorHandler` with
the `defaultWebErrorHandler` as the *Bean Name*:

```java
@Component("defaultWebErrorHandler")
public class CustomDefaultWebErrorHandler implements WebErrorHandler {
    // Omitted
}
```

### Refining Exceptions
Sometimes the given exception is not the actual problem and we need to dig deeper to handle the error, say the actual
exception is hidden as a cause inside the top-level exception. In order to transform some exceptions before handling 
them, we can register an `ExceptionRefiner` implementation as a *Spring Bean*:
```java
@Component
public class CustomExceptionRefiner implements ExceptionRefiner {
    
    @Override
    Throwable refine(Throwable exception) {
        return exception instanceof ConversionFailedException ? exception.getCause() : exception;
    }
}
```

### Registering Custom Handlers
In order to provide a custom handler for a specific exception, just implement the `WebErrorHandler` interface for that
exception and register it as a *Spring Bean*:
```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CustomWebErrorHandler implements WebErrorHandler {
    
    @Override
    public boolean canHandle(Throwable exception) {
        return exception instanceof ConversionFailedException;
    }

    @Override
    public HandledException handle(Throwable exception) {
        return new HandledException("custom_error_code", HttpStatus.BAD_REQUEST, null);
    }
}

```
If you're going to register multiple handlers, you can change their priority using `@Order`. Please note that all your custom
handlers would be registered after built-in exception handlers (Validation, `ExceptionMapping`, etc.). If you don't like
this idea, provide a custom *Bean* of type `WebErrorHandlers` and the default one would be discarded.

### Enabling Web MVC Test Support
In order to enable our test support for `WebMvcTest`s, just add the `@AutoConfigureErrors` annotation to your test
class. That's how a `WebMvcTest` would look like with errors support enabled:
```java
@AutoConfigureErrors
@RunWith(SpringRunner.class)
@WebMvcTest(UserController.class)
public class UserControllerIT {
    
    @Autowired private MockMvc mvc;
    
    @Test
    public void createUser_ShouldReturnBadRequestForInvalidBodies() throws Exception {
        mvc.perform(post("/users").content("{}"))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[0].code").value("username.required"));    
    }
}
```
 
## License
Copyright 2018 alimate

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
