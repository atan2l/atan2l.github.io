---
layout: post
title: "Building an eID-compatible authentication server with Rust"
date: 2025-06-29 14:03:20 +0200
categories: rust web
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

Something, something [The Torment Nexus](https://knowyourmeme.com/memes/torment-nexus).

