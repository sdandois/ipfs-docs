# Discover IPFS content from web2

Below is the suggested heuristic for detecting content-addressed resources on the web, and presenting additional user interface for interacting with that content.

The goal of this page is to provide some suggestions and best practices for web browsers and other [user agents](https://en.wikipedia.org/wiki/User_agent) that are interested in adding support for [content-addressed resources on the Web](/how-to/address-ipfs-on-web).

## Detecting content-addressed resources

In many cases IPFS resources can be detected just by looking at the URI, especially if it is an HTTP URL pointing at a public gateway. This allows a browser to upgrade the transport protocol to IPFS, and remove the need for sending an HTTP request to a remote server.

The high level heuristic is to check, in order:
- Does the `URL` follow [path or subdomain gateway](/how-to/address-ipfs-on-web/) convention?
- If not, does [DNSLink](/concepts/dnslink/) with an IPFS path exist for `URL.hostname`?

If any of the above is true, a user agent should be able to find a content path that can be used for data retrieval over IPFS. See the illustration below.

```mermaid
flowchart TD
    Start[Top-level navigation event]
    IsGateway{Is it a Gateway URL?<br/><code>foo.tld/ipxs/id/..</code><br/><code>id.ipxs.foo.com/..</code>}

    Start -->|new URL in address bar| IsGateway
    IsGateway ==>|YES| IsPathOrSubdomain
    IsPathOrSubdomain -->|Path Gateway| ExtractFromPath
    IsPathOrSubdomain -->|Subdomain Gateway| ExtractFromSubdomain

    subgraph GwURLToPath [Convert Gateway URL to a Content Path]
        IsPathOrSubdomain{Is the Gateway URL 'path' <br/> or 'subdomain' based?}
        ExtractFromPath[Extract <br/> <code>URL.pathname</code>]
        ExtractFromSubdomain[Read namespace <br/> and <br/>identifier from <br/> <code>URL.hostname</code> and  <br/> prepend to <code>URL.pathname</code>. <br/> E.g., turn <br/><code>id.ipxs.foo.com/pathname</code> <br/> into <code>/ipxs/id/pathname</code>]
    end

    PotentialContentPath[Found a Potential Content Path]

    ExtractFromPath --> PotentialContentPath
    ExtractFromSubdomain --> PotentialContentPath
    PotentialContentPath -->|/ipxs/id/sub/path..| IsIpfsOrIpns

    subgraph PathValidation [Is the Content Path valid?]
        IsIpfsOrIpns{Is it <code>/ipfs/</code> <br/>or <code>/ipns/</code><br/>?}
        ValidateRootCID{Is <code>id</code> <br/>a valid CID?}
        ValidateIPNSKey{Is <code>id</code> <br/>a valid CID <br/> with <code>libp2p-key</code> <br/>codec?}
        ValidateDNSLink{Is <code>id</code> <br/> a DNSLink name <br/>  with TXT record: <br/> <code>dnslink=/ipfs/</code><br/> or <code>=/ipns/</code>}

        IsIpfsOrIpns --->|/ipfs/id/..| ValidateRootCID
        IsIpfsOrIpns -->|/ipns/id/..| ValidateIPNSKey
        ValidateIPNSKey -->|NO| ValidateDNSLink
    end

    subgraph FoundValidIPFSContent [Confirmed  we've found a Valid IPFS Resource]
        style FoundValidIPFSContent margin-top: 50
        ValidIPFSCID[Valid /ipfs/cid/.. <br/>Content Path]
        ValidIPNSKey[Valid /ipns/key/.. <br/>Content Path]
        ValidDNSLink[Valid /ipns/dnslink/.. <br/>Content Path]

        ValidateRootCID ===>|YES, /ipfs/cid/..| ValidIPFSCID
        ValidateIPNSKey ===>|YES, /ipns/key/..| ValidIPNSKey
        ValidateDNSLink ===>|YES, /ipfs/dnslink/..| ValidDNSLink
    end

    subgraph NonGwURLToPath [See if non-Gateway URL has a Content Path]
        IsCachedDNSLink{Does <br/> <code>URL.hostname</code> <br/> match a cached <br/> DNSLink name?}
        IsHeaderPresent{Is <code>X-Ipfs-Path</code> <br/> header present?}
        IsError{Did request fail? <br/> HTTP error>500 <br/> or network error}
        IsDNSLinkAtHostname{Does DNSLink exists <br/> for <code>URL.hostname</code>?<br/>}
        PathFromHeader[Read value from <code>X-Ipfs-Path</code>]
    end

    IsGateway -->|NO| IsCachedDNSLink
    IsCachedDNSLink ==>|YES| ValidDNSLink
    IsCachedDNSLink -->|NO| IsHeaderPresent
    IsHeaderPresent ==>|YES| IsDNSLinkAtHostname
    IsHeaderPresent -->|NO| IsError
    IsError ==>|YES| IsDNSLinkAtHostname
    IsDNSLinkAtHostname ==>|YES| ValidDNSLink
    IsDNSLinkAtHostname -->|NO| PathFromHeader
    PathFromHeader --> PotentialContentPath
```

## What to do with detected paths?

### Immutable `/ipfs/cid/..`

- Display "Open via IPFS" button in UI
  - Clicking it should open `ipfs://cid/path?query#hash` (preserving any `?query` or `#hash` from the original HTTP URL)
- If "IPFS Gateway Redirect / Protocol Upgrade" feature is enabled, and the HTTP URL was a gateway one, redirect automatically to `ipfs://cid/path?query#hash`

### Mutable `/ipns/key/..`
- Display "Open via IPFS" button in UI
  - Clicking it should open `ipns://key/path?query#hash` (preserving any `?query` or `#hash` from the original HTTP URL)
- If "IPFS Gateway Redirect / Protocol Upgrade" is enabled, and the original HTTP URL was a gateway one, redirect automatically to `ipns://dnslink/path?query#hash`

### Mutable `/ipns/dnslink/..`

- Display "Open via IPFS" button in UI
  - Clicking it should open `ipns://dnslink/path?query#hash` (preserving `?query` or `#hash` from the original HTTP URL)
- If "DNSLink Website Redirect / Protocol Upgrade" is enabled, redirect automatically to `ipns://dnslink/path?query#hash`
- It is a good practice to internally cache the fact that domain has a valid DNSLink.
  - TTL from TXT record can be used as a hint when to expire cache entry.
  - Performance can be improved further by using cached flag and revalidating it asynchronously, without blocking browser navigation.

## FAQ

- What if a browser does not support `ipfs://` and `ipns://` natively?
  - Implementation can use a [HTTP Gateway](/reference/http/gateway/) as a fallback: convert them to `//gatewayhost/ipxs/id/..` paths, or leverage the built-in URI router at `//gatewayhost/ipfs/?uri=%s`
- Why should `?query` or `#hash` from the original HTTP URL be preserved?
  - Link could point at specific `#section` of  a longer article. It is also common for JS running on a page to use `?query` and `#hash` for navigation and ad-hoc storage for some state.
- Should user agent redirect when URL does not match gateway convention, `URL.hostname` does not have a valid DNSLink, but `X-Ipfs-Path` is present in HTTP response AND points at immutable `/ipfs/cid`?
  - This is an edge case, and we've seen that it is often a misconfiguration caused by invalid/missing DNSLink that could lead to bad UX when automatic redirect is involved: user ends up on immutable copy of a website, bookmark it or keep tab open, and miss updates when DNSLink setup is fixed.
  - It is suggested to not redirect this edge case, or provide a setting that controls this type of redirect. Usually, showing "Open via IPFS" in the user interface is enough for this case.

## Implementation examples

### Brave

[IPFS is supported in Brave since 2021](https://brave.com/brave-integrates-ipfs/). Current features include `ipfs://` and `ipns://` URI support, ability to resolve them using a public or local gateway, opt-in Gateway and DNSLink redirects, and "Open via IPFS" button in the address bar:

> ![Open via IPFS in Brave addressbar](https://user-images.githubusercontent.com/157609/110859368-9a0d7300-82bb-11eb-934d-4e38718dbacb.png)

### IPFS Companion

Firefox and Chromium-based browsers such as Google Chrome, Microsoft Edge can be augumented with [IPFS Companion](/install/ipfs-companion/) browser extension.

It provides detection of IPFS resources, redirect functionality, and a handy UI:

> ![Quick runthrough of basic IPFS Companion features](https://gateway.ipfs.io/ipfs/QmSsGphTN1eWMhkFFNFb23jWTXyhNbo47PF9FbmC6ZaRNg)

