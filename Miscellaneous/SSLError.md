If you see this error running a script
```
(Caused by SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: self-signed certificate (_ssl.c:1001)')))
```

You need to add `verify = False` option to every `request.get()

example
![[Pasted image 20250430182642.png]]
