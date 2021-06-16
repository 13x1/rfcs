- Feature Name: cargo_alternative_registry_auth
- Start Date: 2021-03-31
- RFC PR: rust-lang/rfcs#3139
- Tracking Issue: rust-lang/rust#0000

# Summary
Enables Cargo to include the authorization token for all API requests, crate downloads and index updates (when using HTTP) by adding a configuration option to `config.json` in the registry index.

# Motivation
Organizations need a way to securely publish and distribute internal Rust crates. The current available methods for private crate distribution are awkward: **git repos** do not work well with `cargo update` for resolving semver-compatible dependencies, and do not support the registry API. **Alternative registries** do not support private access and must be operated behind a firewall, or resort to encoding credentials in URLs.

There are many multi-protocol package managers: Artifactory, AWS CodeArtifact, Azure Artifacts, GitHub Artifacts, Google Artifact Registry, and CloudSmith. However, only CloudSmith and Artifactory support Cargo, and they resort to encoding credentials in the URL or allowing anonymous download of packages. This RFC (especially when combined with the approved http-registry RFC) will make it significantly easier to implement Cargo support on private package managers.

# Guide-level explanation
Alternative registry operators can set a new key `auth-required = true` in the registry's `config.json` file, which will cause Cargo to include the Authorization token for all API requests, crate downloads, and index updates (if over HTTP).

```json
{
    "dl": "https://example.com/index/api/v1/crates",
    "api": "https://example.com/",
    "auth-required": true
}
```

If the index is hosted via HTTP using [RFC2789](https://github.com/rust-lang/rfcs/pull/2789) and Cargo receives an `HTTP 401` error when fetching `config.json`, Cargo will automatically re-try the request with the Authorization token included.


# Reference-level explanation
A new key, `auth-required`, will be allowed in the [`config.json`](https://doc.rust-lang.org/cargo/reference/registries.html#index-format) file stored in the registry index. When this key is set to `true`, the authorization token will be sent with any HTTP requests made to the registry API, crate downloads, and index (if using http). If a token is not available when Cargo is attempting to make a request, the user would be prompted to run `cargo login --registry NAME` to save a token.

The authorization token would be sent as an HTTP header, exactly how it is currently sent for operations such as `publish` or `yank`:
```
Authorization: <token>
```

## Interaction with HTTP registries
The approved (but currently unimplemeneted) [RFC2789](https://github.com/rust-lang/rfcs/pull/2789) enables Cargo to fetch the index over HTTP. When fetching `config.json` from an HTTP index, if Cargo receives an `HTTP 401` response, the request will be re-attempted with the Authorization header included. If no authorization token is available, Cargo will suggest that the user run `cargo login` to add one.

To avoid the overhead of an extra HTTP request when fetching `config.json`, the user can optionally configure Cargo locally by setting `auth-required = true` in the `[registries]` table. If the local `auth-required` flag is `true` then Cargo will always include the Authorization token fetching `config.json` over HTTP -- skipping the initial unauthorized requiest and `HTTP 401`. The local configuration option does not impact other operations, such as API requests or downloads. It also does not impact git-based registries.

```toml
[registries]
my-registry = { index = "sparse+https://example.com/index", auth-required = true }
```

## Security
If the server responds with an HTTP redirect, the redirect would be followed, but the Authorization header would *not* be sent to the redirect target.

The authorization header would only be included for requests using `https://`. Under no circumstances would cargo pass an authorization header over an unencrypted `http://` connection. If cargo detected an alternative registry was configured to send the authorization token over an insecure channel, it would exit with an error.

## Interaction with `credential-process`
The unstable [credential-process](https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#credential-process) feature stores credentials keyed on the registry api url, which is only available in after fetching `config.json` from the index. If access to the index is secured using the authorization token, then Cargo will be unable to fetch the `config.json` file before calling the credential process.

For example, the following command would need to download `config.json` from the index before storing the credential.
`cargo login --registry my-registry -Z http-registry -Z credential-process`

To resolve this issue, the credential process feature would use the registry *index url* as the key instead of the *api url*.

Since the token may be used multiple times in a single cargo session (such as updating the index + downloading crates), Cargo should cache the token if it is provided by a `credential-process` to avoid repeatedly calling the credential process.


## Command line options
Cargo commands such as `install` or `search` that support an `--index <INDEX>` command line option to use a registry other than what is available in the configuration file would gain a `--token <TOKEN>` command line option (similar to `publish` today). If a `--token <TOKEN>` command line option is given, the provided authorization token would be sent along with the request.

# Prior art
[prior-art]: #prior-art

The proposed **private-registry-auth** RFC [also proposes](https://github.com/jdemilledt/rfcs/blob/master/text/0000-private-registry-auth.md) sending the authorization token with all requests, but is missing detail.

**NuGet** first attempts to access the index anonymously, then attempts to call credential helpers, then prompts for authentication.

**NPM** uses a local configuration key [`always-auth`](https://docs.npmjs.com/cli/v7/using-npm/config#always-auth). When set to `true` the authorization token is sent with all requests.

**Gradle / Maven (Java)** uses a [local configuration option](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.repositories.MavenArtifactRepository.html) for private package repositories that causes an authorization header to be sent.

**git** first attempts to fetch without authentication. If the server sends back an HTTP 401, then git will send a username & password (if available), or invoke configured [credential helpers](https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage).

# Drawbacks
[drawbacks]: #drawbacks

* There is not a good way to add the authorization header when downloading the index via `git`, so the index authorization will continue to be handled by `git`, until the http-registry RFC is completed.
* Requires a breaking change to the unstable `credential-process` feature, described above under "Interaction with `credential-process`".

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design provides a simple mechanism for cargo to send an authorization header to a registry that works similar to other package managers. Additionally it would work with [RFC2789](https://github.com/rust-lang/rfcs/pull/2789) to serve the index over HTTP, including using a standard web server with basic authentication, since the `token` could be set to `Basic <base64_encoded_credentials>`.

Alternatives:
* Don't add any configuration options to `config.json` or the `[registries]` table and rely on the auto-detection method for everything by first attempting an unauthenticated request, then on HTTP 401, the request would be re-tried including the token. This carries more risk of the token being sent when the server may not be expecting it, but would avoid a configuration option for the registry operator. It also would require more HTTP requests, since each type of request would need to be first attempted without the token.
* Don't add a configuration option to `config.json` and rely only on the local configuration in the `[registries]` table. This avoids the auto-detection, but requires configuration from the user, which could be set up incorrectly or missed.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Do registries need a way to specify which API requests require authorization? Is one switch sufficient?

# Future possibilities
[future-possibilities]: #future-possibilities

## Credential Process
The `credential-process` system could be extended to support generating tokens rather than only storing them. This would further improve security and allow additional features such as 2FA prompts.

## Authentication for Git-based registries
Private registries may want to use the same Authorization header for controlling access to a git-based index over `https`, rather than letting git handle the authentication separately.

This could be enabled by the same local configuration key `auth-required = true` in the `[registries]` table. Both `libgit2` and the `git` command line have a mechanism for specifying an additional header that could be used to pass the Authorization header.