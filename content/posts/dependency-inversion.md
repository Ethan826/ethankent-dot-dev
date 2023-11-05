+++
title = 'Dependency Inversion'
date = 2023-11-04T18:54:36-04:00
draft = false
+++

Among the coding principles you hear get tossed around, Dependency Inversion
sounds boring and is hard to understand. I didn't understand it until a
year or two ago. But it is one of the most valuable tools for organizing your
software and making it easy to change.

One of the hipster moves in programming is to tell people they don't need to use
a framework, especially for backend work. People who say stuff like that don't
typically give concrete guidance on what you should do instead. One of the
clearest advantages of a framework is that it provides an opinionated way to
organize things in a consistent fashion that has worked in production. So if
you're like me, you absorbed that meme about how using a framework makes you a
little bit dumb, but you didn't know what to do instead---or you made a mess.

In particular, organizing code can seem like organizing photos or music files
into folders (don't call me "Boomer"). It seems like a matter of naming files
reasonably and creating folders that make sense. But that turns out to be a
shallow understanding of organizing code.

The Dependency-Inversion Principle, and the "Dependency Rule" it implies, are
powerful tools for organizing code.

## The SOLID Principles are about making change easier.

The Dependency-Inversion Principle ("DIP") is the _D_ in the SOLID Principles.

I get the sense that people see the SOLID Principles as a bit fusty, maybe the
domain of people with pleated pants working in cubicles, inclined to name things
`StaticFactoryBeanFactoryFactory`, and wake up mumbling the names of Go4
patterns. But in reality, they’re good rules of thumb for designing code in a
way that makes it easier to change the things that are likeliest to change. So
the SOLID Principles are as applicable to functional programming as they are to
OOP, to polymorphism through composition as to inheritance-based polymorphism,
and to the 2020s as to the 1990s.

Think of it this way: if you keep a stack of books and papers on your desk, it's
best if they're organized so your daily planner and pocket dictionary are near
the top, and the manual for an old car and the teacher-conference schedule from
three years ago are towards the bottom. Otherwise, you will have to put your
hands on things that are less relevant to get to things that are more so. This
is the Dependency Rule in a nutshell: depend in the direction of stability.

The DIP, and the Clean/Onion/Hexagonal/Ports-and-Adapters Architecture, are
simply elaborations on this idea.

## Introducing an example

Let's start with this code, which creates a presigned POST URL in S3, meaning a
temporary URL you can pass on to a client to let them upload something—say, a
photo—without a password. The `Key` value will be a UUID. We want to put that in
a database table to associate it with the user later (so we can figure out all
the S3 buckets where we put that user's photos).

There are several things wrong with this code. There's no error handling, it
does too many things in a single function, it connects to the database on every
call rather than sharing a connection or pool, etc. But I want you to consider
two things as you read it.

1. How hard is it to test this code?
2. How hard is it to change this code?

```ts
import { randomUUID } from "node:crypto";

import { S3Client } from "@aws-sdk/client-s3";
import { createPresignedPost } from "@aws-sdk/s3-presigned-post";
import { Client } from "pg";

/**
 * Given a user ID, create a presigned post URL and other data and write that
 * data to the database for future use. Return the presigned post data.
 */
export const createAndAssociatePresignedUrl = async (
  userId: number
): Promise<{
  url: string;
  fields: Record<string, string>;
}> => {
  const uuid = randomUUID();
  console.log(`Created UUID: ${uuid}`);

  const s3Client = new S3Client({ region: "us-west-2" });

  const { url, fields } = await createPresignedPost(s3Client, {
    Bucket: "photos",
    Key: uuid,
    Fields: { acl: "public-read" },
  });

  console.log(`Created presigned post: ${JSON.stringify({ url, fields })}`);

  const dbClient = new Client();
  await dbClient.connect();

  console.log("Connected to DB");

  await dbClient.query(
    "INSERT INTO users_photos (user_id, photo_uuid) VALUES ($1, $2);",
    [userId, uuid]
  );

  console.log("Inserted record into `users_photos`");

  return { url, fields };
};
```

The question about testing might seem to come from left field. But the same
things that make code testable tend to make it loosely coupled and therefore
easier to change.

Why is the code hard to test? It's because we're creating or accessing instances
of `console`, the PostgreSQL `Client` class, the `S3Client` class, and the
functions `randomUUID()` and `createPresignedPost()` in our function. To test
these instances we need to use Jest's heavy-handed monkey-patching-based mocking
setup. This approach is annoying and brittle. (For the record I'm experimenting
with Bun, so I'm not actually using Jest with my example code.)

### Dependency-injected version

We can improve this code by injecting these dependencies. We could use a bunch
of positional arguments, but when that starts to get out of hand, an object can
be a better approach. We can also make injecting the dependencies a curried
argument, meaning `(anArg) => (anotherArg) => {}`.

```ts
interface CreateAndAssociatePresignedUrlDeps {
  // This is an example of the Interface-Segregation Principle
  readonly logger: Pick<typeof console, "log">;
  readonly pgClientCtor: typeof Client;
  readonly presignedPostCreator: typeof createPresignedPost;
  readonly s3ClientCtor: typeof S3Client;
  readonly uuidCreator: typeof randomUUID;
}

export const createAndAssociatePresignedUrl = ({
    logger,
    pgClientCtor,
    presignedPostCreator,
    s3ClientCtor,
    uuidCreator,
  }: CreateAndAssociatePresignedUrlDeps): Promise<{
    url: string;
    fields: Record<string, string>;
  }> => async (userId: number) =>
  {
    const uuid = uuidCreator();
    logger.log(`Created UUID: ${uuid}`); // etc. with other `logger` statements

    // ... snip ...

    const s3Client = new s3ClientCtor({ region: "us-west-2" });

    const { url, fields } = await presignedPostCreator(s3Client, {

    // ... snip ...

    const dbClient = new pgClientCtor();

    // ... snip ...
  };
```

This solves the testability problem, which proves we've gone at least part of
the way towards making our code more loosely coupled. Here are some tests.

```ts
import { S3Client } from "@aws-sdk/client-s3";
import { describe, expect, it, jest } from "bun:test";
import { Client } from "pg";

import {
  createAndAssociatePresignedUrl,
  CreateAndAssociatePresignedUrlDeps,
} from ".";

describe("createAndAssociatePresignedUrl", () => {
  const queryMock = jest.fn();
  const logMock = jest.fn();

  // Quick and dirty mocks. All we're trying to do here is to verify that the
  // external APIs are called as expected.
  const mockDeps: CreateAndAssociatePresignedUrlDeps = {
    logger: { log: logMock },
    pgClientCtor: class PgMock {
      query = queryMock;
      connect = () => this;
    } as unknown as typeof Client,
    presignedPostCreator: jest.fn().mockResolvedValue("result"),
    // eslint-disable-next-line @typescript-eslint/no-extraneous-class
    s3ClientCtor: class S3Mock {} as unknown as typeof S3Client,
    uuidCreator: () => "e621e953-8faa-41b4-9ee0-557947ccba3f",
  };

  it("calls the dependencies as expected", async () => {
    expect(await createAndAssociatePresignedUrl(mockDeps)(123)).toEqual({
      url: "testUrl",
      fields: {
        some: "data",
      },
    });

    // Bun doesn't implement `toBeCalledWith` yet.
    expect(logMock.mock.calls).toEqual([
      ["Created UUID: e621e953-8faa-41b4-9ee0-557947ccba3f"],
      ['Created presigned post: {"url":"testUrl","fields":{"some":"data"}}'],
      ["Connected to DB"],
      ["Inserted record into `users_photos`"],
    ]);

    expect(queryMock.mock.calls).toEqual([
      [
        "INSERT INTO users_photos (user_id, photo_uuid) VALUES ($1, $2);",
        [123, "e621e953-8faa-41b4-9ee0-557947ccba3f"],
      ],
    ]);
  });
});
```

Problem solved? Not yet. We've made our code more modular and loosely coupled by
injecting dependencies. But that's not quite the same thing as _inverting_
dependencies. I used to confuse dependency injection with dependency inversion.
The steps we've taken so far are necessary but not sufficient to use the DIP.

## Dependency-inverted version

Remember our two questions from earlier? We've made testing easier, but we're
not in a good position to switch from `console` to Pino or PostgreSQL to
MongoDB. We'd have to tear out most of our code and start again to accomplish
that kind of thing.

### Using "services" and "providers" to invert dependencies

The problem is we're relying too heavily on concrete things: the specific NPM
library `pg`, the specific database PostgreSQL, the specific logger `console`,
etc. What if we were more abstract, more declarative, more domain-focused? What
if we defined the functionality we're using in terms of why and how we're using
it?

For example, we don't want `console.log` per se. If a business stakeholder
stopped us in the hallway and said, "What are you working on today," and we
responded, "Our team is focused on making sure that `console.log` works
properly," the business stakeholder's eyes would glaze over. But if we said,
"Our team is making sure our program can log information so we can see if it's
working correctly," the business stakeholder would nod knowingly and go play a
round of golf or buy a Tesla or whatever it is they get up to.

#### `LoggingService` and its providers

So let's define a capability, not an in-the-weeds implementation.

```ts
export interface LoggingService {
  readonly log: (message: string) => void;
}
```

Hang on a second, isn't that basically `console`? Yes, more or less.

```ts
export const ConsoleLoggingProvider: LoggingService = {
  log: (message: string) => {
    console.log(message);
  },
};
```

But check this out. This is also a valid `LoggingService`.

```ts
const pinoInstance = pino();

export const PinoLoggingProvider: LoggingService = {
  log: (message: string) => {
    // Can't use destructuring with a Pino instance because it's implemented
    // weirdly for performance reasons.
    pinoInstance.info(message);
  },
};
```

So is this. (We're doing various gross things with the instances, transports,
etc., but you get the idea.)

```ts
import winston from "winston";

const { info } = winston;

export const WinstonLoggingProvider: LoggingService = {
  log: (message: string) => {
    info(message);
  },
};
```

To write code that can accept any of these, I have to focus on accepting a
`LoggingService` rather than particular loggers. I'd do this.

```ts
export const heavilyInstrumentedAddition =
  ({ log }: LoggingService) =>
  (num1: number, num2: number) => {
    log("About to do some great math");

    const result = num1 + num2;

    log(`Hey, did you know ${num1} + ${num2} = ${result}?`);

    return result;
  };
```

Now our function doesn't care how `LoggingService` is implemented as long as the
implementation satisfies the type. To choose which instance, we're still using
dependency injection. Let's write our code to use `Pino`

```ts
const pinoVersion = heavilyInstrumentedAddition(PinoLoggingProvider);
pinoVersion(3, 5); // => returns 8
// logs:
// {"level":30,"time":1698808523505,"pid":9973,"hostname":"ethans-mbp.lan","msg":"About to do some great math"}
// {"level":30,"time":1698808523506,"pid":9973,"hostname":"ethans-mbp.lan","msg":"Hey, did you know 3 + 5 = 8?"}
```

Remember our "is it easy to update" question. Is this easy enough for you?

```ts
const consoleVersion = heavilyInstrumentedAddition(ConsoleLoggingProvider);
consoleVersion(3, 5); // => returns 8
// logs:
// About to do some great math
// Hey, did you know 3 + 5 = 8?
```

Notice that we put the dependencies first so that we can partially apply our
function and build instances with different dependencies. This is generally the
better approach—unless you're willing to take the dive into the extremely cool
functional programming concept of the Reader monad. I'll write about that some
time soon.

#### How `LoggingService` and its providers achieve dependency inversion

Oh hey, let's check in on the stultifying-on-first-read definition of the DIP
(lifted from Wikipedia). Now we at least have the context to grok it.

> - High-level modules should not import anything from low-level modules. Both
>   should depend on abstractions (e.g., interfaces).
> - Abstractions should not depend on details. Details (concrete
>   implementations) should depend on abstractions.

Do you see that we’re doing exactly this? Notice that
`heavilyInstrumentedAddition()` doesn't say the name _Pino_, `console`, or
_Winston_? Instead, we name the abstraction/interface: `LoggingService`. And
`LoggingService` does not know Pino, `console`, or Winston, either. There are
now three things:

1. Code that needs to log.
2. Code that describe the ability to log abstractly.
3. Code that actually logs, but matches the abstract description from 2.

By adding in 2., 1. and 3. are now loosely coupled. These terms—_loose
coupling_, _abtraction_, _concretion_, _depend on_—all sound really... abstract.
But look at the example and see that by throwing in the middleman interface and
using that as the glue to hold real code together, we make it very easy to swap
things in and out.

### Defining more services and providers

So let's define some more services. These are just sketches, and as we build
this we'll probably find problems. They also have fewer methods than real
services would have (though the Interface-Segregation Principle does counsel
caution in making services too big).

```ts
/** Represents the ability to interact with bucket-based storage */
export interface BucketService<A = PresignedUrlData> {
  readonly createPresignedUrl: () => Promise<A>;
}

/**
 * Represents the response from successfully creating a presigned URL for an
 * upload.
 */
export interface PresignedUrlData {
  readonly url: string;
  readonly fields: Readonly<Record<string, string>>;
}

/** Represents the ability to set and retrieve data about users in the system */
export interface UserDataService {
  readonly associateKeyWithUser: (userId: number, key: string) => Promise<void>;
}
```

###

Let's start implementing these.

```ts
export const S3BucketProvider: BucketService = {
  createPresignedUrl: async (): Promise<PresignedUrlData> => {
    const s3Client = new S3Client({ region: "us-west-2" });

    const uuid = randomUUID();

    const { url, fields } = await createPresignedPost(s3Client, {
      Bucket: "photos",
      Key: uuid,
      Fields: { acl: "public-read" },
    });

    return { url, fields };
  },
};
```

This provider does satisfy the contract, but we're back to the
testability/difficulty of change issue because we're no longer injecting our
dependencies.

If we look carefully, we'll see two kinds of dependencies: those that correspond
to the S3 Bucket concern and those that don't. It can be a valid design choice
not to inject dependencies related to the concerns the "Provider" fulfills:
after all, this one's entire job is to implement the capabilities of a
`BucketService` for S3 in particular. It is unlikely to be a valid design choice
to use non-domain-related capabilities like UUID generation, for example.

I argue, however, that injecting all of these dependencies is still the better
practice. We can achieve this in various ways, but it makes good sense to expose
a constructor function for a provider that accepts the dependencies that that
provider requires. These dependencies can themselves include services.

```ts
export interface UuidService {
  readonly createUuid: () => string;
}

export interface CreateS3BucketProviderDeps {
  readonly s3Client: S3Client;
  readonly loggingService: LoggingService;
  readonly presignedPostCreator: typeof createPresignedPost;
}

export const createS3BucketProvider = ({
  s3Client,
  loggingService: { log },
  presignedPostCreator,
}: CreateS3BucketProviderDeps): BucketService => ({
  createPresignedUrl: async (key: string): Promise<PresignedUrlData> => {
    const { url, fields } = await presignedPostCreator(s3Client, {
      Bucket: "photos",
      Key: key,
      Fields: { acl: "public-read" },
    });

    log(`Created presigned post: ${JSON.stringify({ url, fields })}`);

    return { url, fields };
  },
});
```

Notice that I just whipped up a `UuidService`, defining it to do what I want.
This situation is common: you may find yourself wishing you had another service.
Well, go ahead and define it—it's a great time because you have a clear
understanding of its use case. Indeed, this is the ideal way to define services:
focusing on the service's users. You'll just need to implement it later. This is
a kind of type-driven development.

```ts
export const NodeUuidProvider: UuidService = {
  // Note this is thunked/called to remove the capability to pass it options,
  // which we either want to disallow or define at the service layer
  createUuid: () => randomUUID(),
};
```

Another provider, this one for `BucketService` using S3.

```ts
export interface CreateS3BucketProviderDeps {
  readonly s3Client: S3Client;
  readonly uuidService: UuidService;
}

export const createS3BucketProvider = ({
  s3Client,
  uuidService: { createUuid },
}: CreateS3BucketProviderDeps) => ({
  createPresignedUrl: async (): Promise<PresignedUrlData> => {
    const uuid = createUuid();

    return createPresignedPost(s3Client, {
      Bucket: "photos",
      Key: uuid,
      Fields: { acl: "public-read" },
    });
  },
});
```

Hold up, we've lost our logging. Well, no problem. We can set these providers up
to require an instance of a `LoggingService`. Notice I said `LoggingService`,
not a logging provider. This point is key: depend on abstractions not
concretions, remember? It's also an illustration of the Dependency Rule: the
service is more stable than the provider.

```ts
export interface CreateS3BucketProviderDeps {
  // ... snip ...
  readonly loggingService: LoggingService;
  // ... snip ...
}

export const createS3BucketProvider = ({
  // ... snip ...
  loggingService: { log },
}: CreateS3BucketProviderDeps) => ({
  createPresignedUrl: async (): Promise<PresignedUrlData> => {
    // ... snip ...

    const { url, fields } = await createPresignedPost(s3Client, { // ...

    // ... snip ...

    log(`Created presigned post: ${JSON.stringify({ url, fields })}`);

    return { url, fields };
  },
});
```

We'd update all our providers in this way.

Let's do our last one, the `UserDataService` for PostgreSQL.

```ts
export interface CreatePostgresUserDataService {
  readonly pgClient: Client;
  readonly loggingService: LoggingService;
}

export const createPostgresUserDataService = ({
  pgClient,
  loggingService: { log },
}: CreatePostgresUserDataService): UserDataService => ({
  associateKeyWithUser: async (userId, uuid) => {
    log(`Preparing to insert user ${userId} with key ${uuid}`);

    await pgClient.query(
      "INSERT INTO users_photos (user_id, photo_uuid) VALUES ($1, $2);",
      [userId, uuid]
    );
  },
});
```

### Defining our main function and fine-tuning the services and providers

Now let's define our main function. First, I'll update the dependencies.

```ts
export interface CreateAndAssociatePresignedUrlDeps {
  readonly loggingService: LoggingService;
  readonly userDataService: UserDataService;
  readonly bucketService: BucketService;
}

export const createAndAssociatePresignedUrl =
  ({
    loggingService,
    userDataService,
    bucketService,
  }: CreateAndAssociatePresignedUrlDeps): Promise<{
    url: string;
    fields: Record<string, string>;
  }> =>
  async (userId: number) => {

  // ... snip ...
```

Whoops. Looking at our original code, we see that we have to use the UUID that's
currently being created inside `BucketService.createPresignedUrl()` in the call
to `UserService.associateKeyWithUser()`. This discovery is an example of the
positive knock-on effects that can happen with well-factored code. Creating a
UUID inside `BucketService.createPresignedUrl()` wasn't really the right design:
the UUID should be an argument for that method, and the main function should
create it.

```ts
interface BucketService<A = PresignedUrlData> {
  // Add `key` as an argument
  readonly createPresignedUrl: (key: string) => Promise<A>;
}

export interface CreateS3BucketProviderDeps {
  // removed dependency on UuidService
  // ... snip ...
}

export const createS3BucketProvider = ({
  // ... snip ...

  // Add the `key` parameter to make it match the service signature
  createPresignedUrl: async (key: string): Promise<PresignedUrlData> => {
    const { url, fields } = await createPresignedPost(s3Client, {
      Bucket: "photos",
      // Use the `key` here
      Key: key,
      Fields: { acl: "public-read" },
    });
```

Now we're ready to define our main function. Notice that it still relies
entirely on services. This indirection may seem like a lot of overhead, but
injecting dependencies even in our main function meets our two part test of
making tests easier to write and dependencies easier to swap out.

```ts
export interface CreateAndAssociateBucketDeps {
  readonly bucketService: BucketService;
  readonly loggingService: LoggingService;
  readonly userDataService: UserDataService;
  readonly uuidService: UuidService;
}

export const createAndAssociateBucket =
  ({
    bucketService: { createPresignedUrl },
    loggingService: { log },
    userDataService: { associateKeyWithUser },
    uuidService: { createUuid },
  }: CreateAndAssociateBucketDeps) =>
  async (
    userId: number
  ): Promise<{
    url: string;
    fields: Record<string, string>;
  }> => {
    const uuid = createUuid();
    log(`Created UUID: ${uuid}`);

    const { url, fields } = await createPresignedUrl(uuid);

    log(`Created presigned post: ${JSON.stringify({ url, fields })}`);

    await associateKeyWithUser(userId, uuid);

    log("Inserted record into `users_photos`");

    return { url, fields };
  };
```

Now we're ready to use this. We'll create an object with our prod dependencies
and partially apply our main function. This is a light-weight factory function.

```ts
const prodDeps: CreateAndAssociateBucketDeps = {
  bucketService: createS3BucketProvider({
    loggingService: ConsoleLoggingProvider,
    s3Client: new S3Client({ region: "us-east-1" }),
    presignedPostCreator: createPresignedPost,
  }),
  loggingService: ConsoleLoggingProvider,
  userDataService: createPostgresUserDataService({
    pgClient: new Client({ region: "us-east-1" }),
    loggingService: ConsoleLoggingProvider,
  }),
  uuidService: NodeUuidProvider,
};

const prodCreateAndAssociateBucket = createAndAssociateBucket(prodDeps);

const { url, fields } = await prodCreateAndAssociateBucket(12345);
```

Swapping out our dependencies is as easy as ensuring we have a provider and then
updating the `prodDeps` object. For example, we could simply change
`ConsoleLoggingProvider` to `PinoLoggingProvider`.

### Testing our solution

Notice that our services and constructing our main function each have their own
dependencies. This approach makes it easier to test at the correct level of
abstraction. Our earlier test suite for the dependency-injected version required
us to mock the specific dependencies, e.g., the PostgreSQL client and the S3
client.

Now, in our main function, we can mock at the level of abstraction that our
services represent. That is, we can make `BucketService.createPresignedUrl()`
return the value we want without actually mocking AWS S3 SDK methods. That
happens to also be the proper level of abstraction for testing the main
function's business logic.

But at the provider level, we get in the weeds, ensuring that the provider
behaves correctly given our various dependencies' return values.

This ability to use mocks at a level of abstraction that matches the
functionality we're trying to test supports test isolation. We don't have to
arrange complicated and brittle test mocks to check each code path in the
high-level functions. And we don't have to find a way to make assertions about
low-level dependencies in a giant function with a zillion moving pieces.

For example, if I want to ensure that we're handling a database connection
failure (spoiler: we aren't), I can test the various failure modes in the
`S3BucketProvider` tests. I can then independently test that the main function
handles any failure from its `BucketService`.

Here's an "in the weeds" test of a provider.

```ts
describe("createS3BucketProvider", () => {
  it("gracefully handles errors thrown by `presignedPostCreator`", () => {
    const loggingSpy = jest.fn();
    const sendSpy = jest.fn();

    const mockDeps = {
      s3Client: { send: sendSpy, config: {} } as unknown as S3Client,
      loggingService: { log: loggingSpy },
      presignedPostCreator: jest.fn().mockRejectedValue("kablooie"),
    };

    // This test will FAIL till we do correct error handling
    expect(
      async () =>
        await createS3BucketProvider(mockDeps).createPresignedUrl("testData")
    ).not.toThrow();
  });
});

// error: expect(received).not.toThrow()

// Thrown value: "kablooie"
```

Here's a "high level" test of the main function.

```ts
describe("createAndAssociateBucket", () => {
  test("happy path", async () => {
    const createPresignedUrlSpy = jest.fn().mockResolvedValue({
      url: "www.fake.com",
      fields: { some: "fields" },
    } satisfies PresignedUrlData);

    const logSpy = jest.fn();

    const associateKeyWithUserSpy = jest.fn().mockResolvedValue(undefined);

    const mockDeps: CreateAndAssociateBucketDeps = {
      bucketService: {
        createPresignedUrl: createPresignedUrlSpy,
      },
      loggingService: {
        log: logSpy,
      },
      userDataService: {
        associateKeyWithUser: associateKeyWithUserSpy,
      },
      uuidService: {
        createUuid: jest
          .fn()
          .mockReturnValue("5940cf0d-5bb3-43a5-bbc0-7801ef23daf3"),
      },
    };

    const subject = await createAndAssociateBucket(mockDeps)(123);

    expect(subject).toEqual({
      url: "www.fake.com",
      fields: {
        some: "fields",
      },
    });

    // Again, I'm using bun:test, which doesn't have `expect().toBeCalledWith()`
    expect(createPresignedUrlSpy.mock.calls).toEqual([
      ["5940cf0d-5bb3-43a5-bbc0-7801ef23daf3"],
    ]);

    expect(logSpy.mock.calls).toEqual([
      ["Created UUID: 5940cf0d-5bb3-43a5-bbc0-7801ef23daf3"],
      [
        'Created presigned post: {"url":"www.fake.com","fields":{"some":"fields"}}',
      ],
      ["Inserted record into `users_photos`"],
    ]);

    expect(associateKeyWithUserSpy.mock.calls).toEqual([
      [123, "5940cf0d-5bb3-43a5-bbc0-7801ef23daf3"],
    ]);
  });
});
```

We can begin changing the mocks' return values to test various failure cases
directly without having to induce the dependencies of dependencies to do what we
want.

## Conclusion

We've seen the value of Dependency Inversion, how it relates to Dependency
Injection, and how it leads to a "Services and Providers" (or similar)
architecture.

These same ideas exist in a variety of frameworks. For example,
Nest.js and Angular use a runtime dependency-injection framework. I believe the
JVM has a variety of runtime solutions. On the opposite end of the spectrum,
tools like `fp-ts`'s Reader monad make this possible with type-level support at
compile time. Effect-TS and ZIO use the Reader pattern along with the ability to
do runtime resolution.

But don't confuse frameworks that help you do this stuff with the ideas
themselves. Any language that supports interface-like abstractions can do this.
This pattern is also the basis of the Clean/Onion/Hexagonal/Ports-and-Adapters
Architecture.

If you understand why and how to use this pattern, you're well on your way to
writing more loosely coupled code and having a mental model of arranging the
dependencies within your codebases.
