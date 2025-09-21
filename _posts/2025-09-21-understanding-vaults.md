---
layout: post
author: kagia
tags: ['cybersecurity', 'vaults', 'secrets', 'NextJs', 'AWS']
---

Supply chain attacks succeed because malicious dependencies can easily access secrets stored as environment variables or in unencrypted files. The solution isn't new; it's a fundamental cybersecurity practice that developers have largely ignored for convenience.

# Vaults

Unsecured key material is just asking to be stolen. That's why you should place all your secrets in a vault. But what is a vault?

In the real world, a vault is secure storage that is only accessible to someone with a key or code that opens the vault. In cybersecurity, it's pretty much the same thing, but as a service. Before we continue, you must be wondering:

## How do we secure the key that opens the vault?

In a well-designed system, you don't. Instead, the key is managed for you. It is provided when it is needed and only to those who are authorised to use it. This introduces the next question of identity: how do we know that it is your program and not a malicious actor? There is a solution for this.

## Signed binaries

Programs wishing to access the vault need a trusted entity to attest that they are who they are. Who is this trusted entity, and what is attestation?

## Platform support

Eventually, we have to trust someone or something. You may have a key to your apartment, but you have to trust your landlord doesn't use the spare key to access your apartment without your approval. Your program is often a tenant in some system. The landlord is the system. If your app runs on a Mac, then the macOS operating system is your landlord. Similarly, your cloud provider, AWS, Google, or Microsoft, is your landlord. You just have to trust your landlord.

The platform is able to track what binaries are run, and it can associate these binaries with a signature unique to each binary. It is important to note that a binary here can mean any program, container, or script.

Requests from your binary to the vault are intercepted by the platform. The platform then attaches a secret made from the signature to the request. This way, the vault is then aware of the identity of the requesting program and grants access only when appropriate.

So how do I sign my binary?

## Signed binaries part II

It depends on your platform*. On macOS, you can use the codesign tool. On AWS, it's a little bit different. Basically, your application or program is hosted in a runtime provided and uniquely trusted by AWS. For example, if you run your program on an EC2 instance, that instance can be uniquely identified by AWS. In that case, it is not necessary to sign your binaries, but it is still recommended (see AWS Signer).

## How do we put it together?

If you are deploying to a platform that provides a vault service, use it. It's not feasible here to detail how this can be done for every context and situation. However, I will mention what that may look like, you are using AWS to deploy a NextJS application.

On AWS, I use the AWS Parameter Store or AWS Secrets Manager. Locally open source solutions, such as https://github.com/ByteNess/aws-vault, to use your computer's keychain as a store for AWS credentials. As a bonus, it doesn't provide these keys directly to programs but instead uses AWS STS to generate short-lived tokens for each invocation.

In 2025, there shouldn't be platforms or SDKs that don't support secure secret storage, but this is sadly a reality. If they do, though, it may not be explicitly clear. One thing to look out for is SDKs and services that allow for keys to be provided during runtime. For example, the Auth0 SDK for JavaScript allows you to provide secrets in the constructor.

{% highlight javascript %}

import { Auth0Client } from "@auth0/nextjs-auth0/server";

let cachedClient;

export async function getAuth0Client() {
  if (cachedClient) return cachedClient;

  const secrets = await call_to_your_vault();
  cachedClient = new Auth0Client({secret: secrets.AUTH0_SECRET, ...});

  return cachedClient;
}
{% endhighlight %}

Once you've done this, remove all traces of these secrets from your .env files, .bashrc, or any other unencrypted file. So the next time you `npm install` and malicious code looks for unencrypted secrets, there are none to be found.

Note: It's also a good idea to look at npm/nodejs alternatives such as pnpm, Bun, and Deno that help prevent install-scripts to begin with.

## Hygene

Hygiene is a must. If your code allows execution of untrusted code, that code will inherit the identity of your program. Isolation, especially in container environments, is critical.

For example, if you are building your NodeJS package, doing so in a separate container that has no access to the vault, for example, could prevent the sort of attacks we are seeing in many NodeJS projects.

Another example is separating modules that need access to the key from software modules that don't. This way, even if a malicious dependency works its way into your final output, it may end up in an environment that still has no access to secrets.

## Important notes

- You can use external vault services other than your platform. However, the root key and responsibility of attestation will lie with the platform hosting your program.

- I simplified the inner workings of platforms and trust models in general. In reality, for example, we may not even trust the platform completely and defer some functionality to the hardware level (see TPM).

- Developers should not have persistent access to production environment secrets. Creating separate environments and permissions is still important.

- In security, there is no absolute, no "never", always be on the lookout.
