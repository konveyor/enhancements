---
title: Crane 2.0 Proxy
authors:
  - "@jmontleon"
reviewers:
  - "@shawn-hurley"
approvers:
  - "@shawn-hurley"
creation-date: 2021-12-20
last-updated: 2021-12-20
status: implementable
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane 2.0 Proxy

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined

## Summary 

The Crane 2.0 Proxy effort is focused on creating a proxy to run in clusters 
alongside other Crane 2.0 in cluster features. The proxy will enable access to
source clusters.

## Motivation

The primary motivation is to prevent CORS issues when showing users information
in the UI. We don't expect this to be used with any other piece of crane 2.0.
  
We must also store user credentials so we can act on behalf of a user.

## Non-goals
- Though storing user credentials plays a part in this Enhancement we do not
currently see a requirement to impersonate users.
- Mutating data: At this time we do not see a requirement to modify or mutate
data returned by clusters.

## Proposal

- Store a list of proxies in a Configmap.
- The list will be a namespace/name combination for each connection to be proxied
- This will essentially be a Ref to a secret containing the URL and credentials
- Credentials will not be used directly by the proxy
- Create a proxy service using go httputil
- We will use gin-gonic to mux connections based on request path
- Each request will take the form of `/namespace/name/*proxyPath`
- We will remove the prefix before passing it on to the cluster.
- The proxy will be exposed via a route to enable client communication.
- SSL will be provided by edge termination for the route

## Links
[httputil](https://pkg.go.dev/net/http/httputil)  
[ReverseProxy](https://pkg.go.dev/net/http/httputil#ReverseProxy)  
[gin-gonic](https://github.com/gin-gonic)  
