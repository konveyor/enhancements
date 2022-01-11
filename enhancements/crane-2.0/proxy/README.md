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

- Store a list of proxies in a CR. This can be a shared CR for other components
to assist looking up credentials.
- Within the CR we will store a cluster/namespace/secret combination
- Create a proxy service using go httputil
- A ReverseProxy will be created for each cluster
- When setting up the ReverseProxy we will get the destination URL from a secret
- The secret will be stored in the namespace specified on the CR
- The secret will also contain credentials for requests
- Credentials will not be used directly by the proxy
- We will use gin-gonic to mux connections based on request path
- Each request will take the form of `/cluster/namespace/secret/*request`
- We will remove the prefix before passing it on to the cluster.
- Communication with the pod can be acheived by running it within the same pod
- If necessary we can expose the proxy as a service.

## Links
[httputil](https://pkg.go.dev/net/http/httputil)  
[ReverseProxy](https://pkg.go.dev/net/http/httputil#ReverseProxy)  
[gin-gonic](https://github.com/gin-gonic)  
