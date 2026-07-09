# Wallet Attached Storage specification

> Wallet Attached Storage is a general-purpose permissioned cloud storage API:
> CRUD over an HTTP hierarchy of Spaces, Collections, and Resources, secured by
> object-capability (zCap) authorization. Storage is private by default -- only
> a Space's controller may act on it unless a policy grants otherwise -- and
> requests that are not authorized return "not found" rather than "not authorized",
> so the existence of a resource is never leaked. The specification is layered:
> a minimal conformant server implements only Resource CRUD plus the authorization
> profile, while listing, collection and space management, multi-tenancy, and
> extensions (linksets, policies, metadata, export, backends, query, quotas) are
> each optional.

This repository contains the Wallet Attached Storage specification (in [ReSpec 
Markdown](https://respec.org/docs/#markdown) format)

For LLM consumption, [`llms.txt`](llms.txt) summarizes the spec and links to its
full text, following the [llmstxt.org](https://llmstxt.org/) convention.

## Table of Contents

- [Background](#background)
- [Code of Conduct](#code-of-conduct)
- [Contributing](#contributing)
- [Status](#status)
- [Usage](#usage)
  - [Editing](#editing)
  - [Testing](#testing)

## Background

You can access the latest version of this specification at:

https://w3c-ccg.github.io/wallet-attached-storage-spec/

The W3C Community Group that is working on this specification can be found
here:

https://w3c-ccg.github.io/

## Code of Conduct

W3C functions under a [code of conduct](https://www.w3.org/Consortium/cepc/).

## Contributing

We encourage contributions meeting the
[Contribution Guidelines](https://github.com/w3c-ccg/community/blob/main/CONTRIBUTING.md). 
While we prefer the creation of issues and Pull Requests in the GitHub 
repository, discussions often occur on the
[public-credentials](http://lists.w3.org/Archives/Public/public-credentials/)
mailing list as well.

## Status

This document is a draft technical specification produced by the W3C Credentials
Community Group, with the intent of progressing toward a W3C standard. Feedback
is welcome via the [issue tracker](https://github.com/w3c-ccg/wallet-attached-storage-spec/issues)
or the [public-credentials mailing list](https://lists.w3.org/Archives/Public/public-credentials/).

## Usage

### Editing

The specification source is in [`spec.md`](./spec.md), in
[ReSpec Markdown](https://respec.org/docs/#markdown) format. It is automatically
imported by `index.html` by the browser at view time.

### Testing

For testing locally, you can `npm i -g http-server`, and then:

```
http-server ./
```
