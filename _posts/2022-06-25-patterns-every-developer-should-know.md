---
title: Patterns That Every Developer Should Know
date: 2022-06-25 00:00:00 +/-TTTT
categories: [typescript]
tags: [design-patterns]     ## TAG names should always be lowercase
---
## Introduction

As developers, we are generally tasked with solving business problems, and as part of our work, we will encounter familiar problems and challenges regardless of the domain. These familiar problems often have common solutions, and these solutions are called **design patterns**.

In this post, I will outline a number of patterns that I believe any developer should be familiar with, regardless of language or level of experience. Even if you never have to actually use any of these patterns, it is important to know what they look like and the problems that they solve.

### A brief note on language choice moving forwards

This article uses TypeScript for code examples, which is a change from my usual habit of using C##, which is my main language. The reason for this is that I want my articles to be as helpful to as many people as possible, and TypeScript is probably the most commonly understood language around that has static typing. Moving forward, I will try to use TypeScript on general posts about software design where it makes sense, but the concepts within the articles themselves are transferrable to just about any language. I don't want to be a .NET blogger- I want to be a Software Engineering blogger.

## Pattern ##1 - Singleton

The **Singleton** pattern is probably one of the first design patterns that a developer usually hears about. A Singleton can be defined as a class that only ever provides one instance, and provides a global reference to it.

The Singleton is frequently referred to as an anti-pattern due to its violation of the Single Responsibility Principle, and the fact that it introduces global state into the application, but it still has its uses. If you have a component in your application that only requires one instance- either for performance reasons or logistical, the Singleton pattern may be the right choice.

Using a code example: imagine that we had a class responsible for reading app configuration from a JSON file:

```ts
import json from './app.settings.json';

export default class AppConfiguration {
    username: string;
    password: string;

    constructor() {
        const { username, password } = json;
        this.username = username;
        this.password = password;
    }
}
```

This code is okay, but every single time we create an instance of the `AppConfiguration` class it will read from the `app.settings.json` file, which could become an expensive operation. Another inefficiency with this code is that these app settings will not change while the application is running, so it's pointless to get the values directly from the file every single time. 

We can turn this class into a Singleton to minimise inefficiency:

```ts
import json from './app.settings.json';

export default class AppConfiguration {
    private static instance: AppConfiguration;

    username: string;
    password: string;

    private constructor() {
        const { username, password } = json;
        this.username = username;
        this.password = password;
    }

    static getInstance(): AppConfiguration {
        return this.instance ??= new AppConfiguration();
    }
}
```

Looking at what we've done, we have:
* made the constructor private
* added a private static field representing the instance
* added a public static method responsible for returning the instance

We would then utilise this Singleton class in the following way:

```ts
const appConfiguration = AppConfiguration.getInstance();
```
By utilising the Singleton pattern here, we are able to cut down on unnecessary operations and provide a cleaner interface for accessing state at the application level.

## Pattern ##2 - Builder

The **Builder** pattern is a design pattern that is recommended for the creation of objects that are highly customizable. Let's consider the following code as an example:

```ts
import { LogLevel } from "./rules";
import { LoggerOptions } from "./options";

export default class Logger {
    constructor(private options: LoggerOptions) { }

    logTrace(msg: string) {
        this.log(msg, 'Trace');
    }

    logWarning(msg: string) {
        this.log(msg, 'Warning');
    }

    logError(msg: string) {
        this.log(msg, 'Error');
    }

    log(msg: string, level: LogLevel) {
        if (this.options.logLevels.get(level) !== false) {
            return;
        }

        if (this.options.includeTimestamp) {
            const timestamp = new Date().toISOString();
            msg = `[${level.toUpperCase()}] ${timestamp}: ${msg}`
        }

        console.log(msg);
    }
}
```

It looks like any old logger class, albeit a fairly primitive one. Take note of the constructor: it takes an `options` parameter of type `LoggerOptions`. This, as the name suggests, holds the various configuration options for our logger:

```ts
import { LogLevel } from "../rules";

export default class LoggerOptions {
    includeTimestamp = false;
    logLevels = new Map<LogLevel, boolean>();
}
```

We would then create our logger instance like this:

```ts
const options = new LoggerOptions();
options.includeTimestamp = true;
options.logLevels.set('Trace', false);

const logger = new Logger(options);

logger.logTrace('should log with timestamp');
```

This looks okay, but imagine if our options class started growing in scope as we expanded on the functionality of our logger. We'd end up with mountains of unreadable configuration code before we had even created our logger class. Lots of configuration code isn't necessarily a bad thing, but readability is important so we should make this as reader-friendly as possible.

A recommended approach here would be to use the Builder pattern to construct our options object. The Builder pattern consists of creating a dedicated class responsible for *building* our object of choice internally, and then outputting that object with a `build` method. A Builder implementation for our `LoggerOptions` class would look like this:

```ts
import { DEFAULT_LOG_LEVELS, LogLevel } from "../rules";

import LoggerOptions from "./LoggerOptions";

export default class LoggerOptionsBuilder {
    private options = new LoggerOptions();

    defaultLogLevels(): LoggerOptionsBuilder {
        this.options.logLevels = DEFAULT_LOG_LEVELS;
        return this;
    }

    setLogLevel(level: LogLevel, enabled: boolean): LoggerOptionsBuilder {
        this.options.logLevels.set(level, enabled);
        return this;
    }

    includesTimestamp(setting: boolean = true): LoggerOptionsBuilder {
        this.options.includeTimestamp = setting;
        return this;
    }

    build(): LoggerOptions {
        return this.options;
    }
}
```

A common practice for the Builder pattern is to make your builder methods return the builder object so that you can chain your method calls like this:

```ts
const options = new LoggerOptionsBuilder()
    .defaultLogLevels()
    .setLogLevel('Trace', false)
    .includesTimestamp()
    .build();

const log = new Logger(options);
```

By utilising the Builder pattern, we are able to create a simplified interface for the creation of highly configurable objects and offer utility methods for common operations that may otherwise involve multiple lines of code. 


## Pattern ##3 - Factory

The **Factory** pattern is another common pattern that can be found in most large codebases - especially those of us working in .NET or Java. This pattern is recommended for the creation of objects with complex creation logic and involves delegating the construction of these objects to what is known as a **Creator** class. Let's look at our example code, a rudimentary tax calculator:

```ts
class Item {
    constructor(public sku: string,
        public price: number,
        public exempt: boolean,
        public imported: boolean) { }

    calculateTax(): number {

        if (this.exempt && this.imported) {
            return this.price * 0.05;
        }
        else if (this.imported) {
            return this.price * 0.15;
        }
        else if (this.exempt) {
            return 0;
        }
        else {
            return this.price * 0.10;
        }
    }
}
```
A quick look at the code tells us the following rules for tax calculation:
* All products are subject to a 10% sales tax, unless exempt.
* Imported products are subject to an additional 5% import tax, regardless of if the product is subject to sales tax.

A common refactoring pattern for code that looks like our tax calculation method would be to [replace conditionals with polymorphism](https://refactoring.guru/replace-conditional-with-polymorphism). This means that we would create a number of subclasses representing each valid state and override the method with the correct functionality:

```ts
abstract class Item {
    constructor(public sku: string,
        public price: number) { }

    abstract calculateTax(): number;
}

class ExemptItem extends Item {
    calculateTax(): number {
        return 0;
    }
}

class TaxableItem extends Item {
    calculateTax(): number {
        return this.price * 0.1;
    }
}

class ExemptImportedItem extends Item {
    calculateTax(): number {
        return this.price * 0.05;
    }
}

class TaxableImportedItem extends Item {
    calculateTax(): number {
        return this.price * 0.15;
    }
}
```

The problem with this refactoring is that while we have simplified our tax calculation logic, we have significantly complicated our creation logic. Every time we need to create a new instance of `Item` we will need to do some property checks to decide the correct concrete class, which isn't ideal. What about when we inevitably have another possible state that could be represented? We would need to update every place where this creation logic is being performed.

Luckily, this is where the Factory pattern comes to the rescue. In order to utilise the Factory pattern, we first must create a new class that is responsible for the creation of our `Item` objects, and then add a `create` method:

```ts
class ItemFactory {
    create(sku: string, price: number,
        exempt: boolean, imported: boolean): Item {

        if (exempt && imported) {
            return new ImportedItem(sku, price);
        }
        else if (exempt) {
            return new ExemptItem(sku, price);
        }
        else if (imported) {
            return new ImportedItem(sku, price);
        }
        else {
            return new TaxableItem(sku, price);
        }

    }
}

// we can now create our items like this:
let item = itemFactory.create('abc123', 29.99, false, true);
```
By utilising this approach, we are able to shift the creation logic into a centralised place and keep our Item classes less cluttered with complex conditionals. This is a good thing because:
* It is a good application of the Single Responsibility principle - creation logic isn't being tacked onto somewhere that it isn't required.
* This code is easier to change. If we ever have a situation where a new type of Item is required, we can simply extend the factory class.

One last thing to note here is that this is a relatively basic implementation of the Factory pattern. There are other evolutions you could make if your use-case demands it, but the reasons for using the pattern remain the same. More information can be found [here](https://refactoring.guru/design-patterns/factory-method).

## Pattern ##4 - Facade

The **Facade** pattern is one of the less common patterns you'll hear about by name, but one of the more common you'll actually see in the wild. The Facade is used when you need to create a simplified interface for something, such as a third-party library, an external service, or even a database.

This is actually the first pattern I would recommend for a developer to learn due to its applicability and simplicity. Most of you have probably already used a Facade already in the form of a Repository class, so this pattern shouldn't be too hard to understand. The general idea of this pattern is to only expose what the consuming class actually cares about, and abstract away the rest. Let's go into our example code:

```ts
class UserService {
    constructor(private repository: UserRepository) { }

    create(user: User) {
        if (!user.email.trim()) throw new Error('Email cannot be blank');
        if (!user.name.trim()) throw new Error('Name cannot be blank');

        // write user to the database
        this.repository.create(user);

        // send email confirmation
        const transporter = createTransport({
            service: 'gmail',
            auth: {
                user: 'youremail@gmail.com',
                pass: '<yourpassword>'
            }
        });

        const mailOptions = {
            from: 'youremail@gmail.com',
            to: user.email,
            subject: 'User created',
            text: `Name: ${user.name}, Email: ${user.email}`
        };

        transporter.sendMail(mailOptions, function (error, info) {
            if (error) {
                console.log(error);
            } else {
                console.log('Email sent: ' + info.response);
            }
        });
    }
}
```

If you've been paying attention, you'll see that this class is already using a Facade in the form of our `UserRepository` interface. Looking at this code, we can see another candidate for the facade pattern in our email code: from the perspective of our `UserService` class, we don't care how the email is sent, only that it is in fact sent. We can apply the Facade pattern by creating an `EmailSender` class and **abstracting** our email logic away from our user logic, then injecting the facade in via the constructor:

```ts
interface EmailSenderOptions {
    senderEmail: string;
    provider: {
        service: string,
        auth: {
            user: string,
            password: string
        }
    }
}

class EmailSender {
    private transporter: Transporter<SMTPTransport.SentMessageInfo>

    constructor(private options: EmailSenderOptions) {
        this.transporter = createTransport(this.options.provider);
    }

    send(to: string, subject: string, message: string) {
        this.transporter.sendMail({
            from: this.options.senderEmail,
            to: to,
            subject: subject,
            text: message
        });
    }
}

class UserService {
    constructor(private repository: UserRepository,
        private emailSender: EmailSender) { }

    create(user: User) {
        if (!user.email.trim())
            throw new Error('Email cannot be blank');

        if (!user.name.trim())
            throw new Error('Name cannot be blank');

        // write user to the database
        this.repository.create(user);

        // send email confirmation
        this.emailSender.send(user.email, 'User Created',
            `Name: ${user.name}, Email: ${user.email}`);
    }
}
```
By utilising the Facade pattern here, we are making our code more compliant with the **Single Responsibility** and **Dependency Inversion** principles. Our code is now properly abstracted, more readable and easier to change if we decide to change our implementation for sending emails.

There isn't much else to the Facade pattern - you simply extract your code into it's own class and inject it into your consuming class or method. When it comes to deciding if you should use this pattern- ask yourself if the code you are thinking about turning into a Facade belongs in the consuming class or not. If it doesn't belong, go for it!

## Final words of wisdom

Once you are familiar with these four patterns, you may find yourself wanting to apply them everywhere - it's okay, this is totally normal. After all, when all you have experience with is a hammer, everything will look like a nail!

Design patterns can be misunderstood and their overuse can add unnecessary complexity to a codebase. A rule of thumb before applying a design pattern is to ask yourself if the application of that pattern simplifies the code, or complicates it. If the answer to that question is the latter: leave it.

A common phrase I use across my blog posts and in my general philosophy towards writing code is: readability is more important than cleverness. Write this on a post-it note and stick it to your monitor, because it will save you from the awkward "I just learned design patterns" phase that pretty much every enterprise developer goes through!

## Further Reading

If you would like to know more about design patterns, most people would recommend the class "Gang of Four" book, but I personally would recommend against it in favour of [Head First Design Patterns](https://www.oreilly.com/library/view/head-first-design/0596007124/). If you like comprehensive but extremely dry textbooks, maybe Gang of Four if for you, but the very approachable writing style of the Head First series is better for those of us that struggle with reading boring books.

If websites are more your thing, I could not recommend [refactoring.guru](https://refactoring.guru/) enough as a visual reference, and if you have any cash to spare I would recommend buying his [book on the subject](https://refactoring.guru/design-patterns/book) too.
