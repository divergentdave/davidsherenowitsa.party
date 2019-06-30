---
layout: "post"
title: "Certificate Carver"
date: "2019-06-30 13:53"
---

I'm happy to announce my first project in Rust! I wrote a command line program, Certificate Carver, to scan files for X.509 certificates, and upload them to Certificate Transparency log servers.

The [source code is available on GitHub](https://github.com/divergentdave/certificate_carver), and there are [pre-built executables available for download](https://github.com/divergentdave/certificate_carver/releases/latest) as well. Read on for more on the background behind this, my motivation, and how to use it.

# What is Certificate Transparency?

The Certificate Transparency protocol is a recent addition to the security infrastructure of the web, introduced by Google in 2013. When you make a secure connection on the web, the server on the other end provides a digital certificate, so that your browser can tell it's talking to the genuine server, and not an attacker. These digital certificates are issued by third parties, Certificate Authorities, and browsers trust CAs to handle verifying who owns which domains. The role of Certificate Transparency is to make auditing these Certificate Authorities easier, by providing append-only logs of these certificates. Logs accept any digital certificate issued by a trusted Certificate Authority, and make them publicly available. (Furthermore, the logs create proofs that certificates are never taken out of the log)

These days, major browsers require that web servers prove their certificates are included in CT logs, so this technology is in widespread use. Monitoring services, such as [Facebook's CT monitor](https://developers.facebook.com/tools/ct/) and [Cert Spotter](https://sslmate.com/certspotter/) allow server operators can use to audit their own domains. For security researchers, the CT logs and related search engines ([crt.sh](https://crt.sh/) and [Censys](https://censys.io/)) provide a gold mine of information on how people use security protocols in practice.

# Where Else Are X.509 Certificates Used?

While digital certificates are predominantly used to secure connections to web servers, the same Certificate Authorities often issue digital certificates for other purposes. The next most common use case is [Windows's AuthentiCode](https://en.wikipedia.org/wiki/Code_signing#Implementations), a code signing scheme used in EXE files, DLL files, and other Windows executables. Software authors sign their programs, libraries, and device drivers, and the signature includes a digital certificate issued to the author by a Certificate Authority. Similarly, digitally signed emails and digitally signed PDF files sometimes include publicly trusted certificates that attest to the author's identity.

Digital certificates are also used to sign iOS apps and Firefox add-ons, but these signatures come from certificates issued by separate, private Certificate Authorities, which are only trusted by the iOS app store, or the Firefox add-ons manager, respectively. (As such, most CT logs would not trust these certificates, and would not accept them for logging.)

# Motivation

There is significant overlap between web and non-web uses of publicly trusted digital certificates. Operating systems and PDF readers trust many of the same Certificate Authorities as browsers. Auditors and researchers who monitor these Certificate Authorities thus have an interest in the certificates they issue for non-web uses, and whether those certificates follow the CA's issuance practices.

Uploading these certificates to CT logs was, up until now, a tedious manual process. Sectigo has a [Certificate Submission Assistant](https://crt.sh/gen-add-chain) that prepares the JSON payload needed to submit a certificate, but users need to combine this with other tools to extract and convert certificates, and to submit the JSON payload to a CT log server.

Certificate Carver fills this gap by combining and automating all the steps needed to submit a certificate to CT logs. It uses file carving techniques to extract certificates from a variety of container formats, by searching for certificate headers and footers. It identifies which certificates are publicly trusted, and prepares the JSON payloads mentioned above. Then, it checks whether the certificates were already logged, and submits any new certificates to CT log servers.

I have run this program on my own computer and an EC2 instance to collect AuthentiCode code signing certificates, on the [Official Journal of the European Union](https://eur-lex.europa.eu/oj/direct-access.html) to collect document signing certificates, and on my browser's profile directory, in case it had any web certificates that were not yet logged.

My hopes with this project are that future research papers will benefit from increased visibility into real-world certificate usage, browser vendors will have better data at hand for oversight of Certificate Authorities, and who knows, maybe a mis-issued certificate or cybersecurity attack will be caught as a result.

# Usage

The easiest way to get started is to download a [pre-built executable](https://github.com/divergentdave/certificate_carver/releases/latest). Select the file that corresponds to your operating system, download it, and extract the ZIP file. Open a command prompt, use the `cd` command to change to the directory where you extracted the files, and then run the command `certificate_carver.exe --help` or `./certificate_carver --help` (depending on your operating system) to check that everything is working. Choose a file or directory you would like to scan, and then pass that path as an argument to the program, for example, `certificate_carver.exe C:\Windows\System32\drivers` if you're using Windows, or `./certificate_carver ~/.mozilla/firefox` if you're using macOS or Linux. The program will scan all the files, print summary information about each certificate found, and then print the results of its log submissions.

You will notice that the program creates a directory named `certificate_carver_cache` in the directory where you run it. This directory records which certificates are already logged, so that the next time the program runs, it can skip checking them.

If you would prefer to build the program from source, rather than using the above binaries, first [install Rust](https://rustup.rs/), then [clone the git repository](https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository#_git_cloning), and run `cargo run --release -- /path/to/files`.

# Future Work

I would like to [add support for digitally signed PDF files.](https://github.com/divergentdave/certificate_carver/issues/14) Certificate Carver does not yet support most such files, because it can't handle PDF compression filters. I have written a prototype for this, but I'd like to find a better PDF library, if possible.

Performance could be improved by [parallelizing file carving](https://github.com/divergentdave/certificate_carver/issues/13). File carving is already I/O bound, but parallelizing will hopefully provide a speed-up by pipelining I/O latencies. (If I/O throughput is saturated, further parallelization won't help.) Parallelization will require significant changes to the internal API.

# Conclusion

If you find this interesting, I'd encourage you to download the program, run it on files you have at hand, and see if you can get some new certificates into the public record. Please let me know if you have any difficulties, questions, or suggestions by [opening a GitHub issue](https://github.com/divergentdave/certificate_carver/issues/new).
