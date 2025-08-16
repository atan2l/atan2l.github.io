---
layout: post
title: "Building an eID-compatible authentication server with Rust"
tags: rust web
---

# Introduction

For quite some time I've been curious about the inner workings of online authentication
using European eID cards. More specifically, my own Spanish eID card. I'd go file my
taxes, or obtain a certificate that I've finished X education curriculum and I'd use my
eID card to access the various government websites and they would know it was me. But
how?

This is where I started researching and learning about mutual TLS (mTLS) and public key
infrastructure (PKI). I also started to wonder: how come nobody outside the government
uses this authentication method?

I remember when I started working at my current job, I had to sign a document online,
and the way they supposedly obtained a valid digital signature from me was by sending a
verification text message to my phone. Looking back on it now I find it odd,
considering that most people can digitally sign documents with their Spanish,
government-issued ID card, or in this case, use it to authenticate themselves to the
site where they would sign.

I think the answer to the question above is that, for some reason, there's no
interest. I haven't seen projects in this line anywhere else, either. So I looked into
it.

Before continuing, I think it's worth mentioning and getting out of the way that I
believe internet privacy to be a fundamental part of what makes the internet, _THE
INTERNET_. The motivation behind this project is purely educational and I think we
shouldn't have our personal identities tied to our internet activities, even though
this project could very well be used for that.

Something, something [The Torment Nexus][torment-nexus].

# mTLS Basics

Transport Layer Security (TLS) is the protocol responsible for keeping the data sent
between you and the server hidden from unintended recipients. It's also the protocol
that enables us to present an X509 certificate to the server to authenticate our
identity.

There are multiple versions of the TLS protocol, but I'll be talking about TLS 1.2 as
I haven't managed to get the authentication to work with TLS 1.3, and TLS 1.2 is the
version used in the government servers that provide auth services which I've
investigated.

The TLS 1.2 protocol is defined in [RFC 5246][rfc-5246] and that's where I begun my
search for information about how I would need to tell my server to request a
certificate when receiving an incoming connection.

Scrolling down to page 36 in RFC 5246 we find the message flow to perform a full TLS
handshake:

{% highlight plaintext %}

      Client                                               Server

      ClientHello                  -------->
                                                      ServerHello
                                                     Certificate*
                                               ServerKeyExchange*
                                              CertificateRequest*
                                   <--------      ServerHelloDone
      Certificate*
      ClientKeyExchange
      CertificateVerify*
      [ChangeCipherSpec]
      Finished                     -------->
                                               [ChangeCipherSpec]
                                   <--------             Finished
      Application Data             <------->     Application Data
{% endhighlight %}

In this flow, the messages marked with an asterisk mean that they're optional or
situation-dependent and that they aren't always sent.

I'm not going to go into detail as to what all the exchanged messages mean, as that's
very clearly and well-explained in the RFC document, but I'm going to touch on a couple
of them that are relevant to what we're tying to achieve.

Namely, I'd like to draw your attention to the `CertificateRequest` message the server
sends, and the corresponding `Certificate` message the client sends in response. These
are the key to prompting the user to provide an authentication certificate in their
browser.

Without going into further detail, it's worth pointing out that the `CertificateRequest`
message also tells the browser client whose CA's certificates you can submit for
authentication.

# The Code
Now that we know what we want to do, let's see if we can write a simple server that
requests a certificate from the user and if they provide any we let them pass. For this
we're going to need a TLS library that implements client certificate authentication.
Fortunately Rust's premier TLS library, `rustls` implements this functionality through
the `WebPkiClientVerifier`.



[torment-nexus]: https://knowyourmeme.com/memes/torment-nexus
[rfc-5246]: https://datatracker.ietf.org/doc/html/rfc5246

