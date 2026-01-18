---
title: "The Fortress of the DOM: Implementing Trusted Types in Vanilla JavaScript"
summary: "Cross-Site Scripting (XSS) has remained a fixture of the OWASP Top 10 for decades, but as the web has shifted from server-side rendering to complex, client-side logic, the nature of the threat has evolved."
categories: ["Post","Blog",]
tags: ["DOM-based-XSS","Trusted-Types","CSP"]
#externalUrl: ""
#showSummary: true
date: 2026-01-17
draft: false
---

#### 1. The Persistent Shadow of DOM-XSS

While "Reflected" and "Stored" XSS are well-understood, **DOM-based XSS** represents a more insidious challenge.

In a DOM-based XSS attack, the vulnerability exists entirely in the client-side code. The server may be perfectly secure, but if the JavaScript on the page takes untrusted data (a "source," like a URL parameter) and passes it to a dangerous "sink" (like `element.innerHTML`), the browser will execute malicious code. Because the payload often never reaches the server, traditional Web Application Firewalls (WAFs) and server-side sanitizers are blind to it.

To solve this fundamentally, the security community has moved beyond reactive blacklisting and toward a structural solution: **Trusted Types**. Trusted Types is a browser-level security primitive that transitions the DOM from an "open-by-default" system to a "locked-down" environment where only verified, type-safe objects can reach dangerous sinks.

---

#### 2. Overview of Trusted Types and Their Purpose

Trusted Types is a Content Security Policy (CSP) directive that effectively "deprecates" the use of raw strings in dangerous DOM sinks. In a standard environment, `innerHTML` accepts any string. With Trusted Types enabled, the browser will throw a type error if you try to pass a string to `innerHTML`. Instead, you must pass a specialized object: a **TrustedHTML**, **TrustedScript**, or **TrustedScriptURL**.

By requiring these objects, Trusted Types forces developers to centralize their security logic. Instead of hoping every developer on a team remembers to sanitize every input, Trusted Types ensures that the *only* way to update the DOM is through a pre-defined, audited "Policy."

---

#### 3. Step-by-Step Guide to Setting up Trusted Types

Implementing Trusted Types in a vanilla JavaScript environment involves three distinct phases: Enforcement, Policy Creation, and Implementation.

##### Phase 1: Enforcement via CSP

The first step is to tell the browser to stop accepting strings in dangerous sinks. This is done via the `Content-Security-Policy` HTTP header:

```http
Content-Security-Policy: require-trusted-types-for 'script'; trusted-types my-policy default;

```

* `require-trusted-types-for 'script'`: This activates enforcement for all sinks that can execute code.
* `trusted-types my-policy default`: This defines a "whitelist" of policy names that are allowed to create Trusted Types.

##### Phase 2: Detecting the Support

Before creating a policy, you should check if the user's browser supports the API:

```javascript
if (window.trustedTypes && window.trustedTypes.createPolicy) {
    // Trusted Types is supported
}

```

##### Phase 3: Creating a Policy

A policy is an object that contains a set of "rules" for transforming a string into a Trusted Type.

```javascript
const myPolicy = trustedTypes.createPolicy('my-policy', {
  createHTML: (input) => {
    // Use a library like DOMPurify or a custom logic to clean the string
    return DOMPurify.sanitize(input);
  },
  createScript: (input) => {
    // Logic for validating scripts if necessary
    return input;
  },
  createScriptURL: (input) => {
    // Ensure URLs only point to trusted domains
    if (input.startsWith('https://trusted-cdn.com/')) {
      return input;
    }
    throw new Error('Untrusted Script URL');
  }
});

```

---

#### 4. Implementation in Vanilla JS

Once the policy is created, you can no longer do this:
`elem.innerHTML = "<img src=x onerror=alert(1)>";` // This will throw a TypeError!

Instead, you must use your policy:

```javascript
const userContent = fetchUserBio(); // Potential XSS source
const secureHTML = myPolicy.createHTML(userContent);
elem.innerHTML = secureHTML; // This works!

```

##### The "Default" Policy

For legacy applications where changing every instance of `innerHTML` is impossible, you can define a **default policy**. The browser will automatically invoke the default policy whenever a string is passed to a sink.

```javascript
trustedTypes.createPolicy('default', {
  createHTML: (input) => DOMPurify.sanitize(input)
});

// This will now work without throwing an error because it 
// passes through the default policy first:
elem.innerHTML = "<img src=x onerror=alert(1)>"; 

```

---

#### 5. Practical Examples: Vulnerability vs. Mitigation

To truly appreciate Trusted Types, we must look at how it handles real-world attack vectors.

##### Example A: The URL Parameter Sink

Imagine a "Search Results" page that displays the search term:

```javascript
// VULNERABLE
const query = new URLSearchParams(window.location.search).get('q');
document.getElementById('display').innerHTML = `Results for: ${query}`;

```

An attacker could send a link: `?q=<img src=x onerror=stealCookies()>`. Without Trusted Types, the browser executes `stealCookies()`. With Trusted Types, the string is rejected because it hasn't been processed by a policy, stopping the attack at the browser's entry point.

##### Example B: Dynamic Script Loading

Many SPAs load modules dynamically:

```javascript
// VULNERABLE
const plugin = new URLSearchParams(window.location.search).get('plugin');
const script = document.createElement('script');
script.src = `/plugins/${plugin}.js`; // Attacker could use '../../evil.com/malice'
document.head.appendChild(script);

```

Trusted Types enforcement on `createScriptURL` ensures that the `src` attribute only accepts a `TrustedScriptURL` object. If the attacker tries to inject a third-party domain, the policy logic we wrote in Phase 3 would throw an error before the script tag is even added to the DOM.

---

#### 6. Transitioning Legacy Codebases

The greatest hurdle to Trusted Types is the "broken" state of existing code. If you have a massive vanilla JS app, turning on enforcement will likely break your site immediately.

**The Strategy:**

1. **Reporting Mode:** Use the `Content-Security-Policy-Report-Only` header. This allows you to see all the places where your code would have broken without actually breaking it.
2. **Audit the Logs:** Use the reports to identify which sinks are receiving raw strings.
3. **Refactor or Default:** Refactor high-risk sinks to use explicit policies and use a `default` policy for lower-risk areas until they can be addressed.

---

#### 7. Zero-Trust for the DOM

Trusted Types represents a paradigm shift in web security. It moves the burden of security from the developer's memory to the browser's architecture. In a vanilla JavaScript environment, it provides a "Source of Truth" for all DOM manipulations, ensuring that no untrusted data can ever transition into executable code.

By enforcing `require-trusted-types-for`, creating audited policies, and utilizing the native API, engineers can effectively eliminate DOM-based XSS from their applications. As privacy and security become the defining features of the modern web, Trusted Types is no longer an optional "extra"—it is the foundation of sovereign, host-proof infrastructure.

This Follow up talk conveys the same idea

{{< youtubeLite id="po6GumtHRmU" label=" Trusted types & the end of DOM XSS - Krzysztof Kotowicz " >}}

