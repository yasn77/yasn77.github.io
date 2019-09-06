# Part II

---

# The woes of Deep Packet Inspection and TLS

---

<blockquote cite="https://en.wikipedia.org/wiki/Deep_packet_inspection">
&ldquo;Deep packet inspection (DPI) is a type of data processing that inspects in detail the data being sent over a computer network, and usually takes action by blocking, re-routing, or logging it accordingly&rdquo;
</blockquote>

---

  * Used heavily in enterprise type organisations
  * Mainly to block malware and other nasty stuff
  * Also used to _'monitor'_ employee online activity

---

## Quick overview of PKI / TLS

![TLS Handshake](https://msdnshared.blob.core.windows.net/media/MSDNBlogsFS/prod.evol.blogs.msdn.com/CommunityServer.Blogs.Components.WeblogFiles/00/00/01/38/03/metablogapi/7608.080613_0416_SSLHandshak4.png) <!-- .element width="75%" height="75%" style="border: 0; background: None; box-shadow: None" -->

(Image taken from https://blogs.msdn.microsoft.com/kaushal/2013/08/02/ssl-handshake-and-https-bindings-on-iis/)

Notes: Once client has the server certificate, it checks it against it list of
CA's and extract the servers public key to create a session key and initiate end
to end encryption

---

As more of the Internet moves to TLS, DPI also _intercepts_ and monitors
encrypted traffic


![DPI TLS](https://upload.wikimedia.org/wikipedia/commons/b/b4/SSL_Deep_Inspection_Explanation.svg) <!-- .element style="border: 0; background: white; box-shadow: None" -->

Notes: TLS request is intercepted. The request is made on behalf of the client
by the proxy. The response is re-encrypted using Enterprise CA certificate and
presented to the requester. Requester see's certificate issued and signed by
Enterprise CA, *NOT* of end site.

------

  * Only the Enterprise CA is valid
  * End to End validation is broken
  * Software depending on Certificate validation will fail

---

## Software updates & TLS DPI
### Broken Red Hat Updates


>CA certificate: The certificate authority which issued the SSL server certificate used by the subscription service. This must be installed on a system for the system to use SSL to connect to the subscription service.

https://access.redhat.com/documentation/en-us/red_hat_subscription_management/1/html/rhsm/certs

------

Red Hat relies on validating server certificate against their own CA. The
subscription and update process requires installing CA certificate issued by Red
Hat. <!-- .element: class="fragment" data-fragment-index="1" -->

If the certificate was not signed by the Red Hat CA, then registering to
Subscription service will fail and the server can not be updated <!-- .element: class="fragment" data-fragment-index="2" -->

------

<!-- .element: data-background="https://media.giphy.com/media/10gmiZVt9hsR5C/source.gif" -->

TLS DPI breaks Red Hat server updates!

Not updating your server == BAD
![hacker](https://media.giphy.com/media/YQitE4YNQNahy/giphy.gif) <!-- .element height="50%" width="50%" style="border: 0; background: white; box-shadow: None" -->

---

## So...

  * DPI increases complexity in keeping systems secure
  * Breaks end to end encryption
  * Prevents systems using completely valid mechanisms to ensure they are talking
    to the right systems
