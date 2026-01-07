---
title: "Error compiling oc for OpenShift on Mac"
date: 2020-03-13T17:36:41
draft: false
slug: error-compiling-oc-for-openshift-on-mac
categories:
  - Uncategorized
wordpress_id: 198
---

When I recently switched to a Mac for my personal/work laptop, I ran
into this problem trying to build
[oc](http://github.com/openshift/oc):

```
# github.com/apcera/gssapi
vendor/github.com/apcera/gssapi/name.go:213:9: could not determine kind of name for C.wrap_gss_canonicalize_name
cgo:
clang errors for preamble:
vendor/github.com/apcera/gssapi/name.go:90:2: error: unknown type name 'gss_const_name_t'
       gss_const_name_t input_name,
       ^
1 error generated.
make: *** [build] Error 2
```

This led me to [this helpful
comment](https://github.com/openshift/oc/blob/release-4.6/vendor/github.com/apcera/gssapi/name.go#L5-L18)
that explains the gssapi headers on Mac are outdated. This was fixed
by installing the [heimdal](https://formulae.brew.sh/formula/heimdal)
HomeBrew package: brew install heimdal
