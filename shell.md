The usage of shell

### Remove the `.example` in the filename under a path
```
# find \
    /etc/modsecurity/owasp-modsecurity-crs \
    -type f -name '*.example' \
 | while read -r f; do cp -p "$f" "${f%.example}"; done \
```
